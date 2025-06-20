# Роль Ansible для установки и настройки AWX с Nginx в Docker Compose

Ниже представлена Ansible роль для развертывания AWX (Ansible Web eXecution) в Docker Compose с Nginx в качестве обратного прокси и полной настройкой расписаний.

## Структура роли

```
awx_docker_compose/
├── defaults/
│   └── main.yml
├── files/
│   ├── docker-compose.yml.j2
│   ├── nginx.conf.j2
│   └── awx_schedules.json
├── tasks/
│   └── main.yml
├── templates/
│   └── custom_awx_config.j2
└── vars/
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки Docker
awx_docker_network: awx_network
awx_postgres_data_dir: /var/lib/awx/postgres
awx_project_data_dir: /var/lib/awx/projects

# Настройки AWX
awx_admin_user: admin
awx_admin_password: "changeme"
awx_secret_key: "your-secret-key-here"
awx_timezone: "UTC"

# Настройки Nginx
awx_nginx_port: 80
awx_nginx_ssl_port: 443
awx_nginx_ssl_cert: ""
awx_nginx_ssl_key: ""
awx_domain: "awx.example.com"

# Настройки базы данных
awx_postgres_user: awx
awx_postgres_password: "awxpass"
awx_postgres_database: awx
awx_postgres_port: 5432

# Настройки RabbitMQ
awx_rabbitmq_user: awx
awx_rabbitmq_password: "awxpass"
awx_rabbitmq_vhost: awx

# Настройки расписаний
awx_schedules_file: "awx_schedules.json"
```

### files/docker-compose.yml.j2

```yaml
version: '3'
services:
  postgres:
    image: postgres:12
    container_name: awx_postgres
    volumes:
      - "{{ awx_postgres_data_dir }}:/var/lib/postgresql/data"
    environment:
      POSTGRES_USER: "{{ awx_postgres_user }}"
      POSTGRES_PASSWORD: "{{ awx_postgres_password }}"
      POSTGRES_DB: "{{ awx_postgres_database }}"
    networks:
      - "{{ awx_docker_network }}"

  rabbitmq:
    image: rabbitmq:3.8-management
    container_name: awx_rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: "{{ awx_rabbitmq_user }}"
      RABBITMQ_DEFAULT_PASS: "{{ awx_rabbitmq_password }}"
      RABBITMQ_DEFAULT_VHOST: "{{ awx_rabbitmq_vhost }}"
    networks:
      - "{{ awx_docker_network }}"

  awx:
    image: ansible/awx:latest
    container_name: awx
    depends_on:
      - rabbitmq
      - postgres
    volumes:
      - "{{ awx_project_data_dir }}:/var/lib/awx/projects"
      - "/etc/passwd:/etc/passwd:ro"
      - "/etc/localtime:/etc/localtime:ro"
    environment:
      AWX_ADMIN_USER: "{{ awx_admin_user }}"
      AWX_ADMIN_PASSWORD: "{{ awx_admin_password }}"
      SECRET_KEY: "{{ awx_secret_key }}"
      DATABASE_USER: "{{ awx_postgres_user }}"
      DATABASE_PASSWORD: "{{ awx_postgres_password }}"
      DATABASE_HOST: "postgres"
      DATABASE_PORT: "{{ awx_postgres_port }}"
      DATABASE_NAME: "{{ awx_postgres_database }}"
      RABBITMQ_USER: "{{ awx_rabbitmq_user }}"
      RABBITMQ_PASSWORD: "{{ awx_rabbitmq_password }}"
      RABBITMQ_HOST: "rabbitmq"
      RABBITMQ_PORT: "5672"
      RABBITMQ_VHOST: "{{ awx_rabbitmq_vhost }}"
      TZ: "{{ awx_timezone }}"
    networks:
      - "{{ awx_docker_network }}"

  awx_nginx:
    image: nginx:latest
    container_name: awx_nginx
    ports:
      - "{{ awx_nginx_port }}:80"
      - "{{ awx_nginx_ssl_port }}:443"
    volumes:
      - "./nginx.conf:/etc/nginx/nginx.conf"
      - "{{ awx_nginx_ssl_cert }}:/etc/nginx/ssl/cert.pem"
      - "{{ awx_nginx_ssl_key }}:/etc/nginx/ssl/key.pem"
    depends_on:
      - awx
    networks:
      - "{{ awx_docker_network }}"

networks:
  {{ awx_docker_network }}:
    driver: bridge
```

### files/nginx.conf.j2

```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    upstream awx {
        server awx:8052;
    }

    server {
        listen 80;
        server_name {{ awx_domain }};

        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {
        listen 443 ssl;
        server_name {{ awx_domain }};

        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        location / {
            proxy_pass http://awx;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /websocket/ {
            proxy_pass http://awx;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
        }
    }
}
```

### tasks/main.yml

```yaml
---
- name: Установка зависимостей
  apt:
    name:
      - docker.io
      - docker-compose
      - python3-pip
    state: present
    update_cache: yes

- name: Установка Docker SDK для Python
  pip:
    name: docker
    state: present

- name: Создание директорий для данных
  file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - "{{ awx_postgres_data_dir }}"
    - "{{ awx_project_data_dir }}"

- name: Копирование docker-compose файла
  template:
    src: files/docker-compose.yml.j2
    dest: /opt/awx/docker-compose.yml
    mode: '0644'

- name: Копирование конфигурации Nginx
  template:
    src: files/nginx.conf.j2
    dest: /opt/awx/nginx.conf
    mode: '0644'

- name: Запуск контейнеров AWX
  docker_compose:
    project_src: /opt/awx
    build: no
    state: present
    restart: yes

- name: Ожидание готовности AWX
  uri:
    url: "http://localhost/api/v2/ping/"
    method: GET
    status_code: 200
    timeout: 30
  register: awx_ready
  until: awx_ready.status == 200
  retries: 30
  delay: 10
  ignore_errors: yes

- name: Установка расписаний AWX
  uri:
    url: "http://localhost/api/v2/schedules/"
    method: POST
    body_format: json
    body: "{{ lookup('file', awx_schedules_file) }}"
    user: "{{ awx_admin_user }}"
    password: "{{ awx_admin_password }}"
    force_basic_auth: yes
    status_code: 201
  when: awx_ready.status == 200
```

### files/awx_schedules.json (пример)

```json
[
  {
    "name": "Daily Backup",
    "description": "Daily backup of all projects",
    "unified_job_template": 1,
    "enabled": true,
    "rrule": "DTSTART:20230101T000000Z RRULE:FREQ=DAILY;INTERVAL=1",
    "extra_data": {}
  },
  {
    "name": "Weekly Inventory Sync",
    "description": "Weekly synchronization of all inventories",
    "unified_job_template": 2,
    "enabled": true,
    "rrule": "DTSTART:20230101T000000Z RRULE:FREQ=WEEKLY;INTERVAL=1;BYDAY=MO",
    "extra_data": {}
  }
]
```

## Использование роли

1. Создайте playbook для использования роли:

```yaml
---
- name: Установка и настройка AWX
  hosts: awx_servers
  become: yes
  roles:
    - awx_docker_compose
```

2. Создайте файл с переменными для переопределения дефолтных значений:

```yaml
---
awx_admin_password: "secure_password_here"
awx_secret_key: "your-secret-key-here"
awx_domain: "awx.yourcompany.com"
awx_nginx_ssl_cert: "/path/to/ssl/cert.pem"
awx_nginx_ssl_key: "/path/to/ssl/key.pem"
```

3. Запустите playbook:

```bash
ansible-playbook -i inventory.ini awx_playbook.yml -e @vars.yml
```

## Дополнительные настройки

Для полной настройки AWX после установки вы можете добавить дополнительные задачи в роль:

- Настройка организаций, команд и пользователей
- Настройка проектов и инвентарей
- Настройка шаблонов заданий
- Настройка уведомлений
- Настройка учетных данных

Роль предоставляет базовую установку AWX с Nginx в Docker Compose с возможностью расширения функционала.