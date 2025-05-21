# Ansible роль для настройки CapsMan на оборудовании MikroTik

Ниже представлена базовая структура Ansible роли для настройки CapsMan (CAPsMAN - Controlled Access Point system Manager) на оборудовании MikroTik.

## Структура роли

```
roles/mikrotik_capsman/
├── tasks/
│   ├── main.yml
│   ├── configure_capsman.yml
│   ├── configure_provisioning.yml
│   └── configure_security.yml
├── defaults/
│   └── main.yml
├── vars/
│   └── main.yml
├── templates/
│   └── capsman_config.rsc.j2
└── README.md
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Основные параметры CapsMan
capsman_enabled: true
capsman_interface: "bridge-local"
capsman_country: "russia"
capsman_channel: 2412
capsman_ssid: "MyWiFiNetwork"
capsman_security_profile: "default"

# Параметры безопасности
security_profile:
  name: "default"
  authentication_types: "wpa2-psk"
  passphrase: "SecurePassword123"
  mode: "dynamic-keys"
  unicast_ciphers: "aes-ccm"
  group_ciphers: "aes-ccm"

# Параметры provisioning
provisioning_enabled: true
provisioning_action: "create-enabled"
provisioning_common_name_regexp: "^CAP"
provisioning_master_configuration: "default"
```

### tasks/main.yml

```yaml
---
- name: Include configuration tasks
  include_tasks: configure_capsman.yml
  when: capsman_enabled

- name: Include security configuration
  include_tasks: configure_security.yml
  when: capsman_enabled

- name: Include provisioning configuration
  include_tasks: configure_provisioning.yml
  when: provisioning_enabled
```

### tasks/configure_capsman.yml

```yaml
---
- name: Configure CAPsMAN basic settings
  community.routeros.command:
    commands:
      - /interface wireless cap set enabled=yes
      - /interface wireless cap set bridge={{ capsman_interface }}
      - /interface wireless cap set discovery-interfaces=all

- name: Configure CAPsMAN channel
  community.routeros.command:
    commands:
      - /interface wireless channel add name=channel1 band=2ghz-g/n control-channel-width=20mhz frequency={{ capsman_channel }} skip-dfs-channels=all

- name: Configure CAPsMAN configuration
  community.routeros.command:
    commands:
      - /interface wireless configuration add name=default mode=ap ssid={{ capsman_ssid }} security-profile={{ capsman_security_profile }} channel=channel1 country={{ capsman_country }} datapath.bridge={{ capsman_interface }}
```

### tasks/configure_security.yml

```yaml
---
- name: Configure security profile
  community.routeros.command:
    commands:
      - /interface wireless security-profiles add name={{ security_profile.name }} authentication-types={{ security_profile.authentication_types }} mode={{ security_profile.mode }} wpa2-pre-shared-key={{ security_profile.passphrase }} unicast-ciphers={{ security_profile.unicast_ciphers }} group-ciphers={{ security_profile.group_ciphers }}
```

### tasks/configure_provisioning.yml

```yaml
---
- name: Configure provisioning rules
  community.routeros.command:
    commands:
      - /interface wireless provisioning add action={{ provisioning_action }} common-name-regexp={{ provisioning_common_name_regexp }} master-configuration={{ provisioning_master_configuration }}
```

## Пример playbook

```yaml
---
- name: Configure MikroTik CapsMan
  hosts: mikrotik_routers
  gather_facts: no
  roles:
    - role: mikrotik_capsman
      vars:
        capsman_ssid: "OfficeWiFi"
        capsman_channel: 2437
        security_profile:
          passphrase: "StrongOfficePassword456"
```

## Требования

1. Установленный модуль `community.routeros` для Ansible
2. Учетные данные для доступа к оборудованию MikroTik
3. Подключение к устройству по SSH или API

## Установка зависимостей

```bash
ansible-galaxy collection install community.routeros
```

## Примечания

1. Роль предполагает, что базовые настройки сети (интерфейсы, bridge) уже настроены
2. Для работы CapsMan необходимо, чтобы на контроллере и точках доступа была установлена соответствующая лицензия
3. Параметры безопасности следует настраивать в соответствии с политиками безопасности организации
4. Для production использования рекомендуется использовать vault для хранения паролей

Вы можете расширить эту роль, добавив дополнительные параметры конфигурации, такие как настройки радио, ограничения клиентов, дополнительные профили безопасности и т.д.