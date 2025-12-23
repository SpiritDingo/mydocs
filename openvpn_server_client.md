Установка и настройка OpenVPN на Ubuntu (сервер и клиент)

1. Установка OpenVPN сервера

1.1 Обновление системы

```bash
sudo apt update
sudo apt upgrade -y
```

1.2 Установка OpenVPN и Easy-RSA

```bash
sudo apt install openvpn easy-rsa -y
```

1.3 Настройка Easy-RSA для управления сертификатами

```bash
# Копируем Easy-RSA в домашнюю директорию
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
```

1.4 Настройка переменных Easy-RSA

```bash
nano vars
```

Добавляем/изменяем в конце файла:

```
export KEY_COUNTRY="RU"
export KEY_PROVINCE="Moscow"
export KEY_CITY="Moscow"
export KEY_ORG="YourOrganization"
export KEY_EMAIL="admin@example.com"
export KEY_OU="IT"
export KEY_NAME="server"
```

1.5 Генерация сертификатов

```bash
# Активируем переменные
source vars

# Очищаем предыдущие ключи (если есть)
./clean-all

# Создаем корневой сертификат (CA)
./build-ca

# Создаем сертификат для сервера
./build-key-server server

# Создаем Diffie-Hellman параметры
./build-dh

# Создаем HMAC-ключ для усиления безопасности
openvpn --genkey --secret keys/ta.key
```

1.6 Копируем сертификаты в папку OpenVPN

```bash
cd ~/openvpn-ca/keys
sudo cp ca.crt server.crt server.key ta.key dh2048.pem /etc/openvpn/
```

1.7 Создаем конфигурацию сервера

```bash
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/
sudo gzip -d /etc/openvpn/server.conf.gz
sudo nano /etc/openvpn/server.conf
```

Основные настройки в server.conf:

```
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh2048.pem
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist /var/log/openvpn/ipp.txt
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
keepalive 10 120
tls-auth ta.key 0
cipher AES-256-CBC
auth SHA256
user nobody
group nogroup
persist-key
persist-tun
status /var/log/openvpn/openvpn-status.log
verb 3
explicit-exit-notify 1
```

1.8 Включаем IP-форвардинг

```bash
sudo nano /etc/sysctl.conf
```

Раскомментируйте строку:

```
net.ipv4.ip_forward=1
```

Применяем изменения:

```bash
sudo sysctl -p
```

1.9 Настраиваем iptables/NFTables

Для iptables:

```bash
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

Для систем с nftables:

```bash
sudo nft add table nat
sudo nft add chain nat postrouting { type nat hook postrouting priority 100 \; }
sudo nft add rule nat postrouting oifname eth0 ip saddr 10.8.0.0/24 counter masquerade
```

1.10 Запускаем и включаем автозагрузку OpenVPN

```bash
sudo systemctl start openvpn@server
sudo systemctl enable openvpn@server
```

2. Создание клиентских сертификатов

2.1 Генерация клиентского сертификата

```bash
cd ~/openvpn-ca
source vars
./build-key client1
```

2.2 Создание клиентского конфигурационного файла

```bash
mkdir -p ~/client-configs/files
nano ~/client-configs/base.conf
```

Содержимое base.conf:

```
client
dev tun
proto udp
remote YOUR_SERVER_IP 1194
resolv-retry infinite
nobind
user nobody
group nogroup
persist-key
persist-tun
remote-cert-tls server
cipher AES-256-CBC
auth SHA256
key-direction 1
verb 3
<ca>
</ca>
<cert>
</cert>
<key>
</key>
<tls-auth>
</tls-auth>
```

2.3 Создание скрипта для генерации клиентских конфигов

```bash
nano ~/client-configs/make_config.sh
```

Содержимое скрипта:

```bash
#!/bin/bash

KEY_DIR=~/openvpn-ca/keys
OUTPUT_DIR=~/client-configs/files
BASE_CONFIG=~/client-configs/base.conf

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn
```

Делаем скрипт исполняемым:

```bash
chmod 700 ~/client-configs/make_config.sh
```

2.4 Генерация конфига для клиента

```bash
cd ~/client-configs
./make_config.sh client1
```

3. Установка OpenVPN клиента на Ubuntu

3.1 Установка OpenVPN

```bash
sudo apt update
sudo apt install openvpn -y
```

3.2 Копирование конфигурационного файла

```bash
# Переносим .ovpn файл с сервера на клиент
# Можно использовать scp, sftp или любой удобный способ

# Пример через scp (выполнять на клиенте):
scp user@server_ip:~/client-configs/files/client1.ovpn ~/
```

3.3 Подключение к VPN

```bash
# Способ 1: Через командную строку
sudo openvpn --config ~/client1.ovpn

# Способ 2: Используя NetworkManager GUI
# Просто откройте файл .ovpn в NetworkManager
```

3.4 Автоматическое подключение (опционально)

```bash
sudo cp ~/client1.ovpn /etc/openvpn/client/
sudo systemctl start openvpn-client@client1
sudo systemctl enable openvpn-client@client1
```

4. Управление OpenVPN

4.1 Проверка статуса сервера

```bash
sudo systemctl status openvpn@server
```

4.2 Просмотр логов

```bash
sudo journalctl -u openvpn@server
tail -f /var/log/openvpn/openvpn-status.log
```

4.3 Перезапуск сервиса

```bash
sudo systemctl restart openvpn@server
```

4.4 Добавление новых клиентов

```bash
cd ~/openvpn-ca
source vars
./build-key client2
cd ~/client-configs
./make_config.sh client2
```

5. Безопасность

5.1 Отзыв сертификата клиента

```bash
cd ~/openvpn-ca
source vars
./revoke-full client1
```

5.2 Обновление списка отозванных сертификатов на сервере

```bash
sudo cp ~/openvpn-ca/keys/crl.pem /etc/openvpn/
```

Добавить в server.conf:

```
crl-verify crl.pem
```

6. Решение проблем

6.1 Проверка подключения

```bash
# На сервере
sudo netstat -uap | grep openvpn

# На клиенте
ping 10.8.0.1
```

6.2 Просмотр маршрутов

```bash
ip route show
```

6.3 Проверка интерфейса tun

```bash
ip addr show tun0
```

Примечания:

1. Замените YOUR_SERVER_IP на реальный IP адрес вашего сервера
2. Для облачных серверов (AWS, DigitalOcean и др.) нужно также настроить правила файервола
3. Рекомендуется использовать TCP порт 443 если UDP 1194 блокируется
4. Для продакшн-среды рассмотрите использование дополнительных мер безопасности

Готовые .ovpn файлы можно использовать на любых клиентах OpenVPN (Windows, macOS, Android, iOS).