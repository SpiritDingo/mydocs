Вот переработанная роль для Oracle Linux 8, 9 и Red Hat:

```yaml
---
- name: Detect OS version and include appropriate variables
  block:
    - name: Include variables for Oracle Linux 8
      include_vars: "OracleLinux_8.yml"
      when: 
        - (ansible_distribution == "OracleLinux" and ansible_distribution_major_version == "8")
        or (ansible_distribution == "RedHat" and ansible_distribution_major_version == "8")
      
    - name: Include variables for Oracle Linux 9
      include_vars: "OracleLinux_9.yml"
      when: 
        - (ansible_distribution == "OracleLinux" and ansible_distribution_major_version == "9")
        or (ansible_distribution == "RedHat" and ansible_distribution_major_version == "9")
      
    - name: Fail for unsupported OS versions
      fail:
        msg: "Unsupported OS {{ ansible_distribution }} version {{ ansible_distribution_major_version }}. Only Oracle Linux 8, 9 and Red Hat 8, 9 are supported."
      when: 
        - ansible_distribution not in ["OracleLinux", "RedHat"]
        or ansible_distribution_major_version not in ["8", "9"]

- name: Create YUM repository directory
  become: yes
  ansible.builtin.file:
    path: /etc/yum.repos.d
    state: directory
    mode: '0755'
  when: not ansible_os_family == "RedHat" | bool

- name: Create authentication directory for repo files
  become: yes
  ansible.builtin.file:
    path: /etc/yum
    state: directory
    mode: '0755'
  when: nexus_username | length > 0 and nexus_password | length > 0

- name: Create repository configuration file
  become: yes
  ansible.builtin.template:
    src: "{{ repo_template }}"
    dest: "/etc/yum.repos.d/{{ repo_filename }}"
    mode: '0644'

- name: Download and install GPG key
  become: yes
  ansible.builtin.get_url:
    url: "{{ gpg_key_url }}"
    dest: "/etc/pki/rpm-gpg/{{ gpg_key_file }}"
    mode: '0644'
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: yes
    validate_certs: no
  when: nexus_username | length > 0 and nexus_password | length > 0

- name: Download and install GPG key without auth
  become: yes
  ansible.builtin.get_url:
    url: "{{ gpg_key_url }}"
    dest: "/etc/pki/rpm-gpg/{{ gpg_key_file }}"
    mode: '0644'
    validate_certs: no
  when: nexus_username | length == 0 or nexus_password | length == 0

- name: Import RPM GPG key
  become: yes
  ansible.builtin.rpm_key:
    state: present
    key: "/etc/pki/rpm-gpg/{{ gpg_key_file }}"

- name: Clean YUM cache
  become: yes
  ansible.builtin.command:
    cmd: yum clean all
  args:
    warn: false

- name: Validate repository access
  become: yes
  ansible.builtin.command:
    cmd: yum repolist
  register: yum_repolist_result
  ignore_errors: yes
  changed_when: false
  when: validate_repo | bool

- name: Fail if repository validation failed
  ansible.builtin.fail:
    msg: "Nexus repository validation failed. Please check credentials and network connectivity."
  when: 
    - yum_repolist_result is failed 
    - validate_repo | bool

- name: Upgrade mdatp package
  become: yes
  ansible.builtin.yum:
    name: mdatp
    state: latest

- name: Install mdatp package if not present
  become: yes
  ansible.builtin.yum:
    name: mdatp
    state: present
  when: install_mdatp | default(true) | bool
```

**Сопутствующие изменения, которые нужно сделать:**

1. **Файлы переменных** (пример для Oracle Linux 8):
```yaml
# OracleLinux_8.yml
repo_template: "nexus.repo.j2"
repo_filename: "nexus.repo"
gpg_key_url: "https://your-nexus/repo/keys/RPM-GPG-KEY-nexus"
gpg_key_file: "RPM-GPG-KEY-nexus"
nexus_username: "{{ vault_nexus_username }}"
nexus_password: "{{ vault_nexus_password }}"
validate_repo: true
```

2. **Шаблон репозитория** (`nexus.repo.j2`):
```ini
[nexus-repo]
name=Nexus Repository
baseurl=https://{% if nexus_username and nexus_password %}{{ nexus_username }}:{{ nexus_password }}@{% endif %}your-nexus-url/repo
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/{{ gpg_key_file }}
sslverify=0
```

**Основные изменения:**

1. **Определение ОС**: Используется `ansible_distribution` и `ansible_distribution_major_version`
2. **Менеджер пакетов**: Заменен `apt` на `yum`
3. **GPG ключи**: Используется стандартное расположение `/etc/pki/rpm-gpg/` и модуль `rpm_key`
4. **Реализация аутентификации**: Учетные данные добавляются прямо в URL репозитория
5. **Валидация репозитория**: Используется `yum repolist` вместо `apt update`
6. **Установка пакетов**: Добавлена отдельная задача для установки пакета, если он отсутствует

Роль теперь поддерживает как Oracle Linux, так и Red Hat версий 8 и 9.



Вот полная Ansible роль для настройки репозитория Nexus на Oracle Linux 8, 9 и Red Hat:

Структура роли

```
roles/nexus_repo/
├── tasks
│   └── main.yml
├── templates
│   └── nexus.repo.j2
├── vars
│   ├── OracleLinux_8.yml
│   └── OracleLinux_9.yml
├── defaults
│   └── main.yml
└── handlers
    └── main.yml
```

Файлы роли

roles/nexus_repo/tasks/main.yml

```yaml
---
- name: Check if running on supported OS
  fail:
    msg: "Unsupported OS {{ ansible_distribution }} version {{ ansible_distribution_major_version }}. Only Oracle Linux 8, 9 and Red Hat 8, 9 are supported."
  when: 
    - ansible_distribution not in ["OracleLinux", "RedHat"]
    or ansible_distribution_major_version not in ["8", "9"]

- name: Include OS-specific variables
  include_vars: "{{ item }}"
  loop:
    - "{{ ansible_distribution }}_{{ ansible_distribution_major_version }}.yml"
  rescue:
    - name: Use default variables if OS-specific file not found
      debug:
        msg: "No OS-specific variables found for {{ ansible_distribution }}_{{ ansible_distribution_major_version }}, using defaults"

- name: Create YUM repository directory
  become: yes
  ansible.builtin.file:
    path: /etc/yum.repos.d
    state: directory
    mode: '0755'

- name: Create repository configuration file
  become: yes
  ansible.builtin.template:
    src: "nexus.repo.j2"
    dest: "/etc/yum.repos.d/{{ repo_filename }}"
    mode: '0644'

- name: Download and install GPG key with authentication
  become: yes
  ansible.builtin.get_url:
    url: "{{ gpg_key_url }}"
    dest: "/etc/pki/rpm-gpg/{{ gpg_key_file }}"
    mode: '0644'
    url_username: "{{ nexus_username | default(omit) }}"
    url_password: "{{ nexus_password | default(omit) }}"
    force_basic_auth: yes
    validate_certs: "{{ validate_certs | default(no) }}"
  when: 
    - nexus_username is defined
    - nexus_password is defined
    - nexus_username | length > 0
    - nexus_password | length > 0

- name: Download and install GPG key without authentication
  become: yes
  ansible.builtin.get_url:
    url: "{{ gpg_key_url }}"
    dest: "/etc/pki/rpm-gpg/{{ gpg_key_file }}"
    mode: '0644'
    validate_certs: "{{ validate_certs | default(no) }}"
  when: 
    - nexus_username is not defined or nexus_username | length == 0
    or nexus_password is not defined or nexus_password | length == 0

- name: Import RPM GPG key
  become: yes
  ansible.builtin.rpm_key:
    state: present
    key: "/etc/pki/rpm-gpg/{{ gpg_key_file }}"

- name: Clean YUM/DNF cache
  become: yes
  ansible.builtin.command:
    cmd: "{{ 'dnf' if ansible_distribution_major_version == '9' else 'yum' }} clean all"
  args:
    warn: false

- name: Validate repository access
  become: yes
  ansible.builtin.command:
    cmd: "{{ 'dnf' if ansible_distribution_major_version == '9' else 'yum' }} repolist enabled"
  register: repo_validation_result
  ignore_errors: yes
  changed_when: false
  when: validate_repo | default(true) | bool

- name: Fail if repository validation failed
  ansible.builtin.fail:
    msg: "Nexus repository validation failed. Please check credentials, network connectivity and repository configuration."
  when: 
    - repo_validation_result is failed
    - validate_repo | default(true) | bool

- name: Install mdatp package
  become: yes
  ansible.builtin.yum:
    name: mdatp
    state: "{{ package_state | default('latest') }}"
  when: install_package | default(true) | bool
  notify: refresh package cache
```

roles/nexus_repo/templates/nexus.repo.j2

```jinja2
[nexus-repo]
name=Nexus Repository
baseurl={% if nexus_username is defined and nexus_password is defined and nexus_username | length > 0 and nexus_password | length > 0 %}https://{{ nexus_username }}:{{ nexus_password }}@{% else %}https://{% endif %}{{ repo_base_url }}
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/{{ gpg_key_file }}
sslverify={{ validate_certs | default(0) | int }}
priority={{ repo_priority | default(99) }}
```

roles/nexus_repo/vars/OracleLinux_8.yml

```yaml
---
repo_filename: "nexus-ol8.repo"
gpg_key_file: "RPM-GPG-KEY-nexus-ol8"
repo_priority: 10
```

roles/nexus_repo/vars/OracleLinux_9.yml

```yaml
---
repo_filename: "nexus-ol9.repo"
gpg_key_file: "RPM-GPG-KEY-nexus-ol9"
repo_priority: 10
```

roles/nexus_repo/defaults/main.yml

```yaml
---
# Nexus repository configuration
repo_base_url: "nexus.example.com/repository/yum-main/"
repo_filename: "nexus.repo"
repo_priority: 99

# GPG key configuration
gpg_key_url: "https://nexus.example.com/repository/keys/RPM-GPG-KEY-nexus"
gpg_key_file: "RPM-GPG-KEY-nexus"

# Authentication (should be overridden in vault or group_vars)
nexus_username: ""
nexus_password: ""

# Security settings
validate_certs: no
validate_repo: yes

# Package management
install_package: yes
package_state: latest  # present|latest|absent

# Package specific settings
package_name: "mdatp"
```

roles/nexus_repo/handlers/main.yml

```yaml
---
- name: refresh package cache
  become: yes
  ansible.builtin.command:
    cmd: "{{ 'dnf' if ansible_distribution_major_version == '9' else 'yum' }} clean all"
  args:
    warn: false
```

Пример playbook

install_nexus_repo.yml

```yaml
---
- name: Configure Nexus repository and install packages
  hosts: all
  become: yes
  vars_files:
    - vault.yml  # Contains nexus_username and nexus_password
  
  roles:
    - role: nexus_repo
      vars:
        repo_base_url: "your-actual-nexus-domain/repository/your-repo-name/"
        gpg_key_url: "https://your-actual-nexus-domain/repository/keys/your-gpg-key"
        package_name: "your-package-name"
```

Файл с секретами (vault)

vault.yml (шифруется с помощью ansible-vault)

```yaml
---
nexus_username: "your-nexus-username"
nexus_password: "your-nexus-password"
```

Использование

1. Создайте структуру роли:

```bash
mkdir -p roles/nexus_repo/{tasks,templates,vars,defaults,handlers}
```

1. Создайте все файлы как указано выше
2. Зашифруйте файл с паролями:

```bash
ansible-vault encrypt vault.yml
```

1. Запустите playbook:

```bash
ansible-playbook -i inventory install_nexus_repo.yml --ask-vault-pass
```

Особенности роли:

· Поддержка Oracle Linux 8, 9 и Red Hat 8, 9
· Автоматическое определение менеджера пакетов (yum/dnf)
· Гибкая аутентификация - работает как с учетными данными, так и без них
· Валидация репозитория перед установкой пакетов
· Поддержка самоподписанных сертификатов
· Приоритеты репозиториев для контроля порядка установки пакетов
· Обработчики для обновления кэша пакетов
· Безопасное хранение секретов через ansible-vault

Роль готова к использованию и может быть легко адаптирована под ваши конкретные требования к репозиторию Nexus.
