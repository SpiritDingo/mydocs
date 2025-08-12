Настройка мостового интерфейса (bridge) в Linux позволяет объединить несколько сетевых интерфейсов в один логический, что полезно для виртуализации, маршрутизации и других сетевых задач.  

### **1. Установка необходимых пакетов (если нет)**
```bash
sudo apt update && sudo apt install bridge-utils  # Для Debian/Ubuntu
sudo yum install bridge-utils                   # Для CentOS/RHEL
```

### **2. Создание моста (вручную)**
#### **Способ 1: Через `ip` (временный, до перезагрузки)**
```bash
sudo ip link add name br0 type bridge           # Создать мост br0
sudo ip link set dev br0 up                     # Активировать мост
sudo ip link set eth0 master br0                # Добавить интерфейс eth0 в мост
sudo ip addr add 192.168.1.100/24 dev br0       # Назначить IP мосту (если нужно)
```

#### **Способ 2: Через `nmcli` (NetworkManager)**
```bash
sudo nmcli connection add type bridge ifname br0
sudo nmcli connection add type bridge-slave ifname eth0 master br0
sudo nmcli connection up br0
```

### **3. Постоянная настройка (через конфиги)**
#### **Debian/Ubuntu (`/etc/network/interfaces`)**
```bash
sudo nano /etc/network/interfaces
```
Добавить:
```ini
auto br0
iface br0 inet static
    bridge_ports eth0
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
```
Применить:
```bash
sudo systemctl restart networking
```

#### **CentOS/RHEL (`/etc/sysconfig/network-scripts/ifcfg-br0`)**
```bash
sudo nano /etc/sysconfig/network-scripts/ifcfg-br0
```
Добавить:
```ini
DEVICE=br0
TYPE=Bridge
BOOTPROTO=static
IPADDR=192.168.1.100
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
ONBOOT=yes
```
И изменить конфиг eth0 (`ifcfg-eth0`):
```ini
DEVICE=eth0
BRIDGE=br0
ONBOOT=yes
```
Применить:
```bash
sudo systemctl restart network
```

### **4. Проверка**
```bash
ip addr show br0           # Проверить IP моста
brctl show                 # Показать мосты и их интерфейсы
ping 192.168.1.1           # Проверить связность
```

### **5. Дополнительно**
- **Добавление интерфейсов в мост**:
  ```bash
  sudo brctl addif br0 eth1
  ```
- **Удаление интерфейса из моста**:
  ```bash
  sudo brctl delif br0 eth1
  ```
- **Удаление моста**:
  ```bash
  sudo ip link delete br0
  ```

> **Примечание:**  
> - Если используется `NetworkManager`, лучше настраивать через `nmcli`.  
> - Для виртуализации (KVM, LXC) мост часто используется без IP, а VM получают адреса через DHCP.  
> - Если мост не работает, проверьте, что `netfilter` включен:  
>   ```bash
>   sudo modprobe br_netfilter
>   echo "br_netfilter" | sudo tee /etc/modules-load.d/br_netfilter.conf
>   ```

Теперь у вас настроен мостовой интерфейс в Linux! 🚀




### **Подробное руководство по настройке моста (bridge) в Linux**  

Мостовой интерфейс (`bridge`) объединяет несколько сетевых интерфейсов в единый логический сегмент, что полезно для:  
- Виртуализации (KVM, LXC, Docker)  
- Объединения физических интерфейсов в одну сеть  
- Создания прозрачного межсетевого шлюза  

Рассмотрим настройку вручную (`ip`, `brctl`), через конфигурационные файлы (`netplan`, `NetworkManager`, `ifupdown`) и диагностику.  

---

## **1. Установка необходимых утилит**  
Пакет `bridge-utils` (устарел, но иногда нужен) и `iproute2` (современная замена).  

### **Debian/Ubuntu**  
```bash
sudo apt update
sudo apt install -y bridge-utils iproute2
```

### **RHEL/CentOS**  
```bash
sudo yum install -y bridge-utils iproute
```

### **Arch Linux**  
```bash
sudo pacman -S iproute2 bridge-utils
```

---

## **2. Временная настройка моста (до перезагрузки)**  
### **Создание моста через `ip`**  
```bash
sudo ip link add name br0 type bridge       # Создать мост br0
sudo ip link set br0 up                     # Включить мост
sudo ip addr add 192.168.1.100/24 dev br0   # Назначить IP (опционально)
```

### **Добавление интерфейса в мост**  
```bash
sudo ip link set eth0 master br0            # Добавить eth0 в br0
```

### **Проверка**  
```bash
ip link show br0            # Проверить состояние моста
bridge link show            # Список интерфейсов в мостах
ping 192.168.1.1            # Проверить связь
```

### **Удаление интерфейса из моста**  
```bash
sudo ip link set eth0 nomaster  # Убрать eth0 из моста
```

### **Удаление моста**  
```bash
sudo ip link delete br0
```

---

## **3. Постоянная настройка (конфигурационные файлы)**  
### **3.1. Debian/Ubuntu (`/etc/network/interfaces`)**  
```bash
sudo nano /etc/network/interfaces
```
```ini
# Настройка моста br0
auto br0
iface br0 inet static
    bridge_ports eth0       # Интерфейсы в мосте
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8
```
Применить:  
```bash
sudo systemctl restart networking
```

### **3.2. RHEL/CentOS (`/etc/sysconfig/network-scripts/`)**  
#### **Конфиг моста (`ifcfg-br0`)**  
```bash
sudo nano /etc/sysconfig/network-scripts/ifcfg-br0
```
```ini
DEVICE=br0
TYPE=Bridge
BOOTPROTO=static
IPADDR=192.168.1.100
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
ONBOOT=yes
```

#### **Конфиг интерфейса (`ifcfg-eth0`)**  
```bash
sudo nano /etc/sysconfig/network-scripts/ifcfg-eth0
```
```ini
DEVICE=eth0
BRIDGE=br0
ONBOOT=yes
```
Применить:  
```bash
sudo systemctl restart network
```

### **3.3. Ubuntu (Netplan, актуально с 18.04)**  
```bash
sudo nano /etc/netplan/01-netcfg.yaml
```
```yaml
network:
  version: 2
  renderer: networkd
  bridges:
    br0:
      dhcp4: no
      addresses: [192.168.1.100/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
      interfaces: [eth0]
```
Применить:  
```bash
sudo netplan apply
```

---

## **4. Настройка фильтрации (если трафик не проходит)**  
Иногда мост блокируется `iptables/nftables`.  

### **Включение форвардинга**  
```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

### **Загрузка модуля `br_netfilter`**  
```bash
sudo modprobe br_netfilter
echo "br_netfilter" | sudo tee /etc/modules-load.d/br_netfilter.conf
```

### **Отключение фильтрации в мосте**  
```bash
echo 0 | sudo tee /proc/sys/net/bridge/bridge-nf-call-iptables
```

---

## **5. Диагностика проблем**  
### **Проверка состояния моста**  
```bash
bridge link show        # Список интерфейсов в мосте
ip link show br0        # Статус моста
ip addr show br0        # IP-адрес моста
```

### **Проверка маршрутизации**  
```bash
ip route show          # Таблица маршрутизации
ping 8.8.8.8          # Проверка доступа в интернет
```

### **Логи (`journalctl`)**  
```bash
journalctl -xe -u networking   # Для Debian
journalctl -xe -u network     # Для RHEL
```

---

## **6. Дополнительные настройки**  
### **STP (Spanning Tree Protocol)**  
Включение STP предотвращает петли в сети:  
```bash
sudo brctl stp br0 on       # Включить STP для br0
```

### **Изменение MAC-адреса моста**  
```bash
sudo ip link set dev br0 address 00:11:22:33:44:55
```

### **Добавление VLAN в мост**  
```bash
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip link set eth0.100 master br0
```

---

## **Вывод**  
Теперь у вас есть:  
✅ Временный мост (через `ip`)  
✅ Постоянный мост (через конфиги)  
✅ Настройка Netplan (Ubuntu)  
✅ Решение проблем с фильтрацией  
✅ Диагностика и дополнительные настройки  

Можно использовать мост для виртуализации, объединения интерфейсов или создания сложных сетевых конфигураций. 🚀