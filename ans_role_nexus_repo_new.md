# Ansible роль для миграции репозиториев на Nexus

Роль для миграции с Spacewalk на Nexus репозиторий для Oracle Linux 9 и Ubuntu в корпоративной среде.

## Структура роли

```
roles/nexus_migration/
├── tasks/
│   ├── main.yml
│   ├── oracle_linux.yml
│   ├── ubuntu.yml
│   └── remove_spacewalk.yml
├── templates/
│   ├── oracle_nexus.repo.j2
│   └── ubuntu_nexus.list.j2
├── vars/
│   └── main.yml
└── README.md
```

## Файлы роли

### vars/main.yml

```yaml
---
# Nexus репозиторий URL
nexus_url: "https://nexus.example.com"

# Пути к репозиториям в Nexus
nexus_oracle_repo_path: "/repository/ol9"
nexus_ubuntu_repo_path: "/repository/ubuntu"

# Имена репозиториев
oracle_repo_name: "ol9_nexus"
ubuntu_repo_name: "ubuntu_nexus"
```

### tasks/remove_spacewalk.yml

```yaml
---
- name: Удаление агента Spacewalk для Oracle Linux
  yum:
    name: rhn-client-tools
    state: absent
  when: ansible_distribution == "OracleLinux"

- name: Удаление агента Spacewalk для Ubuntu
  apt:
    name: rhn-client-tools
    state: absent
  when: ansible_distribution == "Ubuntu"

- name: Удаление конфигурации Spacewalk
  file:
    path: "/etc/sysconfig/rhn/"
    state: absent
```

### tasks/oracle_linux.yml

```yaml
---
- name: Удаление старых репозиториев Oracle Linux
  file:
    path: "/etc/yum.repos.d/{{ item }}"
    state: absent
  loop:
    - "oracle-linux-ol9.repo"
    - "spacewalk.repo"

- name: Копирование конфигурации репозитория Nexus для Oracle Linux
  template:
    src: "oracle_nexus.repo.j2"
    dest: "/etc/yum.repos.d/{{ oracle_repo_name }}.repo"
    owner: root
    group: root
    mode: '0644'

- name: Очистка кеша yum
  yum:
    clean_all: yes

- name: Обновление индексов репозиториев
  yum:
    update_cache: yes
```

### tasks/ubuntu.yml

```yaml
---
- name: Удаление старых репозиториев Ubuntu
  file:
    path: "/etc/apt/sources.list.d/{{ item }}"
    state: absent
  loop:
    - "spacewalk.list"
    - "ubuntu-official.list"

- name: Копирование конфигурации репозитория Nexus для Ubuntu
  template:
    src: "ubuntu_nexus.list.j2"
    dest: "/etc/apt/sources.list.d/{{ ubuntu_repo_name }}.list"
    owner: root
    group: root
    mode: '0644'

- name: Обновление индексов apt
  apt:
    update_cache: yes
```

### tasks/main.yml

```yaml
---
- name: Удаление Spacewalk
  include_tasks: remove_spacewalk.yml

- name: Настройка репозиториев для Oracle Linux
  include_tasks: oracle_linux.yml
  when: ansible_distribution == "OracleLinux"

- name: Настройка репозиториев для Ubuntu
  include_tasks: ubuntu.yml
  when: ansible_distribution == "Ubuntu"
```

### templates/oracle_nexus.repo.j2

```ini
[{{ oracle_repo_name }}]
name=Nexus Repository for Oracle Linux 9
baseurl={{ nexus_url }}{{ nexus_oracle_repo_path }}
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
sslverify=0
```

### templates/ubuntu_nexus.list.j2

```plaintext
deb {{ nexus_url }}{{ nexus_ubuntu_repo_path }} {{ ansible_distribution_release }} main restricted
deb {{ nexus_url }}{{ nexus_ubuntu_repo_path }} {{ ansible_distribution_release }}-updates main restricted
deb {{ nexus_url }}{{ nexus_ubuntu_repo_path }} {{ ansible_distribution_release }}-security main restricted
```

## Использование роли

1. Создайте playbook для применения роли:

```yaml
---
- name: Миграция на Nexus репозиторий
  hosts: all
  become: yes
  roles:
    - nexus_migration
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini migrate_to_nexus.yml
```

## Особенности реализации

1. **Корпоративное решение** - роль использует переменные для централизованного управления конфигурацией.
2. **Удаление Spacewalk** - предусмотрено полное удаление клиента Spacewalk.
3. **Локальный репозиторий Nexus** - конфигурационные файлы генерируются из шаблонов.
4. **Nexus уже настроен** - роль предполагает, что Nexus работает и доступен.
5. **Без порта 443** - в URL не указывается порт, используется стандартный HTTPS.
6. **Без CE-сертификата** - отключена проверка SSL (`sslverify=0` для Oracle, аналогично для Ubuntu).

Роль поддерживает как Oracle Linux 9, так и Ubuntu, автоматически определяя ОС и применяя соответствующие настройки.