# Роль Ansible для миграции локального репозитория с Spacewalk на Nexus

Эта роль предназначена для переноса локальных репозиториев с Spacewalk на Nexus для Oracle Linux и Ubuntu систем.

## Структура роли

```
roles/migrate_repo/
├── tasks/
│   ├── main.yml
│   ├── oracle_linux.yml
│   └── ubuntu.yml
├── templates/
│   ├── oraclelinux-nexus.repo.j2
│   └── ubuntu-nexus.list.j2
└── vars/
    └── main.yml
```

## Файлы роли

### vars/main.yml

```yaml
---
# Nexus репозиторий URL
nexus_url: "http://your-nexus-server/repository"
nexus_repo_name: "your-repo-name"

# Spacewalk репозиторий URL (для справки)
spacewalk_url: "http://your-spacewalk-server"

# Для Oracle Linux
oracle_linux_releases:
  - "7"
  - "8"
  - "9"

# Для Ubuntu
ubuntu_releases:
  - "focal"
  - "jammy"
  - "lunar"
```

### tasks/main.yml

```yaml
---
- name: Include OS-specific tasks
  include_tasks: "{{ ansible_distribution | lower }}.yml"
  when: ansible_distribution in ['OracleLinux', 'Ubuntu']
  
- name: Fail if unsupported OS
  fail:
    msg: "Unsupported OS {{ ansible_distribution }}. Only OracleLinux and Ubuntu are supported."
  when: ansible_distribution not in ['OracleLinux', 'Ubuntu']
```

### tasks/oracle_linux.yml

```yaml
---
- name: Remove Spacewalk client
  yum:
    name: rhn-client-tools
    state: absent
  when: "'rhn-client-tools' in ansible_facts.packages"

- name: Remove Spacewalk repo files
  file:
    path: "/etc/yum.repos.d/{{ item }}"
    state: absent
  loop:
    - "rhn.repo"
    - "spacewalk.repo"
  ignore_errors: yes

- name: Install Nexus repo configuration for Oracle Linux
  template:
    src: "oraclelinux-nexus.repo.j2"
    dest: "/etc/yum.repos.d/nexus-{{ nexus_repo_name }}.repo"
    owner: root
    group: root
    mode: '0644'

- name: Clean yum cache
  command: yum clean all

- name: Update all packages
  yum:
    name: '*'
    state: latest
```

### tasks/ubuntu.yml

```yaml
---
- name: Remove Spacewalk client
  apt:
    name: rhn-client-tools
    state: absent
    autoremove: yes
  when: "'rhn-client-tools' in ansible_facts.packages"

- name: Remove Spacewalk repo files
  file:
    path: "/etc/apt/sources.list.d/{{ item }}"
    state: absent
  loop:
    - "rhn.list"
    - "spacewalk.list"
  ignore_errors: yes

- name: Install Nexus repo configuration for Ubuntu
  template:
    src: "ubuntu-nexus.list.j2"
    dest: "/etc/apt/sources.list.d/nexus-{{ nexus_repo_name }}.list"
    owner: root
    group: root
    mode: '0644'

- name: Update apt package index
  apt:
    update_cache: yes

- name: Upgrade all packages
  apt:
    upgrade: dist
```

### templates/oraclelinux-nexus.repo.j2

```jinja2
[nexus-{{ nexus_repo_name }}]
name=Nexus Repository for Oracle Linux $releasever - $basearch
baseurl={{ nexus_url }}/oraclelinux/$releasever/$basearch
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
```

### templates/ubuntu-nexus.list.j2

```jinja2
deb {{ nexus_url }}/ubuntu {{ ansible_distribution_release }} main restricted universe multiverse
deb {{ nexus_url }}/ubuntu {{ ansible_distribution_release }}-updates main restricted universe multiverse
deb {{ nexus_url }}/ubuntu {{ ansible_distribution_release }}-security main restricted universe multiverse
```

## Использование роли

1. Создайте playbook (например, `migrate_repo.yml`):

```yaml
---
- hosts: all
  become: yes
  roles:
    - migrate_repo
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini migrate_repo.yml
```

## Дополнительные замечания

1. Убедитесь, что Nexus уже настроен и содержит необходимые репозитории для Oracle Linux и Ubuntu.
2. Для Oracle Linux могут потребоваться дополнительные GPG ключи.
3. Для Ubuntu может потребоваться настройка аутентификации, если Nexus требует этого.
4. Перед выполнением в production рекомендуется протестировать на тестовых серверах.