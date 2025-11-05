Вот полный пример файла daemon.json для подключения Docker к Nexus registry с авторизацией:

```json
{
  "registry-mirrors": [],
  "insecure-registries": [
    "nexus.local:8082",
    "192.168.1.100:8082",
    "your-nexus-domain:8082"
  ],
  "debug": true,
  "experimental": false,
  "log-level": "info",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "max-concurrent-uploads": 10,
  "max-download-attempts": 5,
  "registry-auth": {
    "nexus.local:8082": {
      "auth": "dXNlcm5hbWU6cGFzc3dvcmQ="
    }
  }
}
```

🔧 Подробное описание параметров:

Основные настройки:

· insecure-registries - список registry, которые используют HTTP вместо HTTPS (обязательно для Nexus по умолчанию)
· nexus.local:8082 - замените на hostname и порт вашего Nexus сервера
· 192.168.1.100:8082 - замените на IP-адрес вашего Nexus сервера

Аутентификация:

Для генерации auth token выполните:

```bash
echo -n 'username:password' | base64
# ИЛИ
echo -n 'admin:your-password' | base64
```

Замените dXNlcm5hbWU6cGFzc3dvcmQ= на полученный base64-код.

🔄 Альтернативный способ настройки аутентификации:

1. Через docker login (рекомендуется):

```bash
docker login nexus.local:8082
# Введите логин и пароль при запросе
```

2. Через файл конфигурации Docker:

Файл ~/.docker/config.json будет содержать:

```json
{
  "auths": {
    "nexus.local:8082": {
      "auth": "dXNlcm5hbWU6cGFzc3dvcmQ="
    }
  }
}
```

⚙️ Дополнительные настройки для продакшн среды:

```json
{
  "insecure-registries": [
    "nexus.company.com:8082"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true",
    "overlay2.size=20G"
  ]
}
```

🔒 Настройка HTTPS (рекомендуется для продакшн):

Если используете HTTPS, уберите insecure-registries и добавьте сертификат:

```bash
# Копирование CA сертификата Nexus
sudo mkdir -p /etc/docker/certs.d/nexus.company.com:8082
sudo cp nexus-ca.crt /etc/docker/certs.d/nexus.company.com:8082/ca.crt
```

🚀 Применение конфигурации:

1. Сохраните файл как /etc/docker/daemon.json
2. Перезапустите Docker демон:

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

1. Проверьте подключение:

```bash
docker login nexus.local:8082
docker pull nexus.local:8082/your-image:tag
```

🛠️ Устранение проблем:

Если возникают ошибки:

· Проверьте, что порт registry доступен: telnet nexus.local 8082
· Убедитесь, что Docker Bearer Token Realm включен в Nexus
· Проверьте логи: journalctl -u docker -f

Эта конфигурация обеспечит надежное подключение Docker к вашему локальному Nexus repository с поддержкой авторизации.