# Ansible роль для подключения точек доступа к CapsMan на MikroTik

Эта роль автоматизирует процесс подключения точек доступа MikroTik к контроллеру CapsMan (CAPsMAN).

## Структура роли

```
roles/mikrotik_ap_provisioning/
├── tasks/
│   ├── main.yml
│   ├── prepare_ap.yml
│   ├── connect_to_capsman.yml
│   └── verify_connection.yml
├── defaults/
│   └── main.yml
├── vars/
│   └── main.yml
├── templates/
│   └── ap_config.rsc.j2
└── README.md
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Основные параметры
capsman_controller_ip: "192.168.88.1"
capsman_controller_port: "8728"
capsman_username: "admin"
capsman_password: ""

# Параметры точки доступа
ap_username: "admin"
ap_password: ""
ap_ssh_port: 22

# Настройки подключения
ap_provisioning_ssid: "CAPsMAN_Provisioning"
ap_reset_to_default: false
ap_wait_timeout: 120
```

### tasks/main.yml

```yaml
---
- name: Prepare access point
  include_tasks: prepare_ap.yml
  when: ap_reset_to_default

- name: Connect AP to CapsMAN
  include_tasks: connect_to_capsman.yml

- name: Verify connection
  include_tasks: verify_connection.yml
```

### tasks/prepare_ap.yml

```yaml
---
- name: Reset AP to factory defaults
  community.routeros.command:
    host: "{{ inventory_hostname }}"
    username: "{{ ap_username }}"
    password: "{{ ap_password }}"
    port: "{{ ap_ssh_port }}"
    commands:
      - /system reset-configuration no-defaults=yes skip-backup=yes

- name: Wait for AP to reboot
  wait_for:
    host: "{{ inventory_hostname }}"
    port: "{{ ap_ssh_port }}"
    timeout: "{{ ap_wait_timeout }}"
```

### tasks/connect_to_capsman.yml

```yaml
---
- name: Configure basic AP settings
  community.routeros.command:
    host: "{{ inventory_hostname }}"
    username: "{{ ap_username }}"
    password: "{{ ap_password }}"
    port: "{{ ap_ssh_port }}"
    commands:
      - /interface wireless set [ find default-name=wlan1 ] disabled=no
      - /interface wireless security-profiles add name=temp mode=none
      - /interface wireless set [ find default-name=wlan1 ] security-profile=temp

- name: Configure provisioning SSID
  community.routeros.command:
    host: "{{ inventory_hostname }}"
    username: "{{ ap_username }}"
    password: "{{ ap_password }}"
    port: "{{ ap_ssh_port }}"
    commands:
      - /interface wireless set [ find default-name=wlan1 ] ssid="{{ ap_provisioning_ssid }}" mode=ap-bridge

- name: Connect to CapsMAN controller
  community.routeros.command:
    host: "{{ inventory_hostname }}"
    username: "{{ ap_username }}"
    password: "{{ ap_password }}"
    port: "{{ ap_ssh_port }}"
    commands:
      - /interface wireless cap set enabled=yes
      - /interface wireless cap set caps-address={{ capsman_controller_ip }}
      - /interface wireless cap set caps-manager=yes
      - /interface wireless cap set discovery-interfaces=wlan1
```

### tasks/verify_connection.yml

```yaml
---
- name: Wait for AP to connect to CapsMAN
  pause:
    seconds: 30
    prompt: "Waiting for AP to connect to CapsMAN..."

- name: Verify connection status on controller
  community.routeros.command:
    host: "{{ capsman_controller_ip }}"
    username: "{{ capsman_username }}"
    password: "{{ capsman_password }}"
    port: "{{ capsman_controller_port }}"
    commands:
      - /interface wireless registration-table print where interface="wlan1"
  register: registration_status
  until: registration_status.stdout | length > 0
  retries: 5
  delay: 10

- name: Display connection status
  debug:
    var: registration_status.stdout
```

## Пример playbook

```yaml
---
- name: Provision MikroTik APs to CapsMAN
  hosts: mikrotik_aps
  gather_facts: no
  roles:
    - role: mikrotik_ap_provisioning
      vars:
        capsman_controller_ip: "10.0.0.1"
        capsman_username: "capsman-admin"
        capsman_password: "!vault|encrypted_password_here!"
        ap_password: "!vault|encrypted_ap_password_here!"
        ap_provisioning_ssid: "CAPsMAN_Join_Network"
```

## Дополнительные возможности

1. **Групповое добавление AP**:
```yaml
- name: Add multiple APs with different names
  community.routeros.command:
    commands:
      - /interface wireless provisioning add action=create-enabled common-name-regexp="^AP-{{ item }}" master-configuration=default
  loop:
    - "floor1"
    - "floor2"
    - "floor3"
```

2. **Настройка параметров радио**:
```yaml
- name: Configure radio settings
  community.routeros.command:
    commands:
      - /interface wireless radio set [ find default-name=wlan1 ] band=2ghz-g/n frequency-mode=superchannel channel-width=20mhz
```

3. **Настройка VLAN**:
```yaml
- name: Configure VLAN on AP
  community.routeros.command:
    commands:
      - /interface wireless set [ find default-name=wlan1 ] vlan-id={{ vlan_id }} vlan-mode=use-tag
```

## Требования

1. Учетные данные для доступа к точкам доступа и контроллеру CapsMan
2. Доступ по SSH/API к устройствам
3. Установленный модуль `community.routeros`

## Рекомендации

1. Используйте Ansible Vault для хранения паролей
2. Перед массовым развертыванием протестируйте на одной точке доступа
3. Убедитесь, что версии RouterOS на контроллере и точках доступа совместимы
4. Для больших сетей рассмотрите возможность использования tags в Ansible для группировки устройств

## Пример использования vault

1. Создайте файл vault:
```bash
ansible-vault create group_vars/all/vault.yml
```

2. Добавьте в него секретные данные:
```yaml
---
capsman_password: "SuperSecret123"
ap_password: "AP_Password456"
```

3. В основном playbook укажите:
```yaml
vars_files:
  - group_vars/all/vault.yml
```