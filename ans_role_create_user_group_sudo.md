# Ansible роль для создания пользователя и группы с правами sudo в Linux

Вот пример Ansible роли, которая создаёт пользователя, группу и предоставляет ей права sudo.

## Структура роли

```
roles/setup_user/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    └── sudoers.j2
```

## 1. defaults/main.yml

```yaml
---
# Имя пользователя
user_name: "deployer"
# Пароль пользователя (в зашифрованном виде)
user_password: "$6$rounds=656000$V4B3L.lXjO1sYIuT$XJ8hD0UzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQ"
# Имя группы
group_name: "sudoers"
# Путь к домашнему каталогу
user_home: "/home/{{ user_name }}"
# Оболочка пользователя
user_shell: "/bin/bash"
# Список SSH ключей для пользователя
user_ssh_keys: []
```

## 2. tasks/main.yml

```yaml
---
- name: Создание группы
  group:
    name: "{{ group_name }}"
    state: present

- name: Создание пользователя
  user:
    name: "{{ user_name }}"
    group: "{{ group_name }}"
    password: "{{ user_password }}"
    home: "{{ user_home }}"
    shell: "{{ user_shell }}"
    create_home: yes
    state: present

- name: Добавление SSH ключей для пользователя
  authorized_key:
    user: "{{ user_name }}"
    key: "{{ item }}"
    state: present
  loop: "{{ user_ssh_keys }}"
  when: user_ssh_keys | length > 0

- name: Настройка прав sudo для группы
  template:
    src: sudoers.j2
    dest: /etc/sudoers.d/{{ group_name }}
    owner: root
    group: root
    mode: 0440
    validate: /usr/sbin/visudo -cf %s
```

## 3. templates/sudoers.j2

```jinja2
# Файл сгенерирован Ansible, не редактировать вручную
%{{ group_name }} ALL=(ALL:ALL) NOPASSWD: ALL
```

## Использование роли

1. Создайте зашифрованный пароль с помощью команды:
   ```bash
   mkpasswd --method=SHA-512 --rounds=656000
   ```

2. Добавьте роль в playbook:

```yaml
- hosts: all
  become: yes
  roles:
    - role: setup_user
      vars:
        user_name: "admin"
        group_name: "admins"
        user_ssh_keys:
          - "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ..."
```

## Безопасность

1. Для реального использования рекомендуется хранить пароли в Ansible Vault.
2. SSH ключи также лучше хранить в зашифрованном виде.
3. Убедитесь, что файл sudoers имеет правильные права (0440).

## Альтернативный вариант с использованием Ansible Vault

Если вы хотите использовать Ansible Vault для хранения пароля:

1. Создайте файл `vars/secrets.yml` и зашифруйте его:
   ```bash
   ansible-vault create vars/secrets.yml
   ```

2. Добавьте в него:
   ```yaml
   user_password: "$6$rounds=656000$V4B3L.lXjO1sYIuT$XJ8hD0UzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQZzQ"
   ```

3. Обновите playbook:
   ```yaml
   - hosts: all
     become: yes
     vars_files:
       - vars/secrets.yml
     roles:
       - role: setup_user
   ```

Эта роль предоставляет гибкую настройку пользователей с правами sudo в Linux системах.