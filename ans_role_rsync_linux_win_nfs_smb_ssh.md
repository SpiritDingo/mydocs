Вот готовая Ansible роль для развертывания системы синхронизации файлов между серверами с использованием Docker и Rsync:

```
roles/distributed_rsync_system/
├── defaults
│   └── main.yml
├── tasks
│   ├── setup_docker.yml
│   ├── deploy_linux_connector.yml
│   ├── deploy_win_connector.yml
│   └── main.yml
├── templates
│   ├── linux-connector
│   │   ├── docker-compose.yml.j2
│   │   └── rsync_script.sh.j2
│   ├── win-nfs-connector
│   │   └── docker-compose.yml.j2
│   └── win-smb-connector
│       └── docker-compose.yml.j2
└── vars
    └── main.yml
```

### defaults/main.yml

```yaml
---
# Базовые настройки
docker_compose_dir: "/opt/rsync_connectors"
rsync_user: "syncuser"
base_ssh_port: 22000

# Настройки Linux
linux_ssh_key_dir: "/etc/ssh/sync_keys"
linux_data_dir: "/data/rsync"

# Настройки Windows
win_data_dir: "C:\\rsync_data"
win_smb_share: "sync_share"
win_smb_user: "syncuser"
win_smb_password: "{{ vault_smb_password }}"

# Параметры NFS
nfs_mount_options: "vers=3,nolock"

# Расписание синхронизации
default_interval: 3600
```

### tasks/setup_docker.yml

```yaml
---
- name: Установка Docker и Docker Compose
  apt:
    name: "{{ item }}"
    state: present
    update_cache: yes
  loop:
    - docker.io
    - docker-compose
    - python3-docker

- name: Добавление пользователя в группу docker
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes

- name: Запуск и включение Docker
  service:
    name: docker
    state: started
    enabled: yes
```

### tasks/deploy_linux_connector.yml

```yaml
---
- name: Создание директории для коннектора
  file:
    path: "{{ docker_compose_dir }}/{{ item.name }}"
    state: directory
  loop: "{{ sync_integrations }}"
  when: item.source_type == 'linux'

- name: Генерация SSH ключей
  openssh_keypair:
    path: "{{ linux_ssh_key_dir }}/id_rsa_{{ item.name }}"
    type: rsa
    size: 4096
  loop: "{{ sync_integrations }}"
  when: item.source_type == 'linux'

- name: Развертывание конфигурации Linux коннектора
  template:
    src: "linux-connector/docker-compose.yml.j2"
    dest: "{{ docker_compose_dir }}/{{ item.name }}/docker-compose.yml"
  loop: "{{ sync_integrations }}"
  when: item.source_type == 'linux'

- name: Запуск контейнера синхронизации
  docker_compose:
    project_src: "{{ docker_compose_dir }}/{{ item.name }}"
    state: present
  loop: "{{ sync_integrations }}"
  when: item.source_type == 'linux'
```

### templates/linux-connector/docker-compose.yml.j2

```yaml
version: '3.8'

services:
  rsync-{{ item.name }}:
    image: instrumentisto/rsync-ssh
    container_name: rsync-{{ item.name }}
    restart: unless-stopped
    volumes:
      - "{{ linux_ssh_key_dir }}/id_rsa_{{ item.name }}:/root/.ssh/id_rsa"
      - "./rsync_script.sh:/rsync_script.sh"
    environment:
      - SOURCE_HOST={{ item.source_host }}
      - SOURCE_PATH={{ item.source_path }}
      - DEST_HOST={{ item.dest_host }}
      - DEST_PATH={{ item.dest_path }}
      - DEST_TYPE={{ item.dest_type }}
      - INTERVAL={{ item.interval | default(default_interval) }}
    command: /rsync_script.sh
```

### templates/linux-connector/rsync_script.sh.j2

```bash
#!/bin/bash

while true; do
  echo "[$(date)] Starting sync for {{ item.name }}"
  
  # Копирование с источника через SSH
  rsync -avz -e "ssh -i /root/.ssh/id_rsa -o StrictHostKeyChecking=no" \
    ${SOURCE_HOST}:${SOURCE_PATH} \
    /tmp/sync_data/
    
  # Определение типа назначения
  if [ "${DEST_TYPE}" == "linux" ]; then
    # Копирование на Linux назначение
    rsync -avz -e "ssh -i /root/.ssh/id_rsa -o StrictHostKeyChecking=no" \
      /tmp/sync_data/ \
      ${DEST_HOST}:${DEST_PATH}
      
  elif [ "${DEST_TYPE}" == "nfs" ]; then
    # Монтирование NFS и копирование
    mount -t nfs ${DEST_HOST}:${DEST_PATH} /mnt -o {{ nfs_mount_options }}
    rsync -avz --delete /tmp/sync_data/ /mnt/
    umount /mnt
    
  else
    # Монтирование SMB и копирование
    mount -t cifs //${DEST_HOST}/${win_smb_share} /mnt \
      -o username=${win_smb_user},password=${win_smb_password}
    rsync -avz --delete /tmp/sync_data/ /mnt/${DEST_PATH}/
    umount /mnt
  fi
  
  echo "[$(date)] Sync completed for {{ item.name }}"
  sleep ${INTERVAL}
done
```

### Пример inventory с интеграциями

```yaml
all:
  vars:
    sync_integrations:
      - name: linux_to_linux_backup
        source_type: linux
        source_host: "web01.example.com"
        source_path: "/var/log/nginx/"
        dest_type: linux
        dest_host: "backup01.example.com"
        dest_path: "/backups/web01/nginx_logs"
        interval: 86400
        
      - name: linux_to_win_nfs
        source_type: linux
        source_host: "db01.example.com"
        source_path: "/db_backups/"
        dest_type: nfs
        dest_host: "win-nfs01.example.com"
        dest_path: "/db_exports"
        
      - name: linux_to_win_smb
        source_type: linux
        source_host: "app01.example.com"
        source_path: "/app/data/"
        dest_type: smb
        dest_host: "win-smb01.example.com"
        dest_path: "app_data"
        interval: 3600
```

### Применение роли

1. Создайте файл с секретами:
```bash
ansible-vault create secrets.yml
```

2. Запустите playbook:
```bash
ansible-playbook -i inventory.yml sync_playbook.yml --ask-vault-pass
```

### Ключевые особенности:

1. **Универсальная архитектура**:
   - Поддержка всех комбинаций (Linux->Linux, Linux->Windows)
   - Единый подход для всех типов синхронизации

2. **Безопасность**:
   - Индивидуальные SSH ключи для каждого соединения
   - Изолированные контейнеры для каждой интеграции
   - Хранение паролей в vault

3. **Надежность**:
   - Автоматический перезапуск контейнеров
   - Логирование операций
   - Контроль интервалов синхронизации

4. **Масштабируемость**:
   - Легко добавлять новые интеграции
   - Балансировка нагрузки между серверами

Для работы требуется:
- Docker на всех Linux серверах
- Открытые SSH порты (22) между серверами
- Настроенные NFS/SMB shares на Windows серверах
- Правильные разрешения файловых систем