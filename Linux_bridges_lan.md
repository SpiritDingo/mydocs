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