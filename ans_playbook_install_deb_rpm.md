# Ansible Playbook для установки deb и rpm пакетов

Вот пример playbook, который может устанавливать пакеты как в системах на базе Debian/Ubuntu (использующих `.deb` пакеты), так и в системах на базе RedHat/CentOS (использующих `.rpm` пакеты):

```yaml
---
- name: Установка пакетов deb и rpm
  hosts: all
  become: yes
  gather_facts: yes  # Важно для определения типа ОС

  vars:
    # Пути к пакетам (можно переопределить при вызове playbook)
    deb_package_path: /path/to/package.deb
    rpm_package_path: /path/to/package.rpm

  tasks:
    - name: Установка deb пакета на Debian/Ubuntu системы
      ansible.builtin.apt:
        deb: "{{ deb_package_path }}"
      when: ansible_facts['os_family'] == 'Debian'

    - name: Установка rpm пакета на RedHat/CentOS системы
      ansible.builtin.yum:
        name: "{{ rpm_package_path }}"
      when: ansible_facts['os_family'] == 'RedHat'

    # Альтернативный вариант для rpm с использованием модуля rpm
    - name: Установка rpm пакета (альтернативный вариант)
      ansible.builtin.rpm:
        package: "{{ rpm_package_path }}"
      when: ansible_facts['os_family'] == 'RedHat'
```

## Как использовать этот playbook

1. Сохраните код выше в файл, например `install_packages.yml`
2. Запустите playbook, указав пути к вашим пакетам:
   ```bash
   ansible-playbook install_packages.yml -e "deb_package_path=/full/path/to/package.deb" -e "rpm_package_path=/full/path/to/package.rpm"
   ```

## Дополнительные варианты

### Установка из URL

Если пакеты доступны по URL, можно модифицировать playbook:

```yaml
- name: Скачать и установить deb пакет
  ansible.builtin.apt:
    deb: "https://example.com/package.deb"
  when: ansible_facts['os_family'] == 'Debian'

- name: Скачать и установить rpm пакет
  ansible.builtin.yum:
    name: "https://example.com/package.rpm"
  when: ansible_facts['os_family'] == 'RedHat'
```

### Установка локальных пакетов с копированием

Если нужно сначала скопировать пакеты на целевые машины:

```yaml
- name: Копировать deb пакет
  ansible.builtin.copy:
    src: "{{ deb_package_path }}"
    dest: /tmp/package.deb
    remote_src: no
  when: ansible_facts['os_family'] == 'Debian'

- name: Установить скопированный deb пакет
  ansible.builtin.apt:
    deb: /tmp/package.deb
  when: ansible_facts['os_family'] == 'Debian'

- name: Копировать rpm пакет
  ansible.builtin.copy:
    src: "{{ rpm_package_path }}"
    dest: /tmp/package.rpm
    remote_src: no
  when: ansible_facts['os_family'] == 'RedHat'

- name: Установить скопированный rpm пакет
  ansible.builtin.yum:
    name: /tmp/package.rpm
  when: ansible_facts['os_family'] == 'RedHat'
```

Playbook автоматически определит тип системы и выполнит соответствующую задачу установки.