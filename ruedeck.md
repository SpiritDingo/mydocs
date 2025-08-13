Вот пример файла `docker-compose.yml` для развертывания Rundeck с использованием Docker:

```yaml
version: '3'

services:
  rundeck:
    image: rundeck/rundeck:latest
    container_name: rundeck
    environment:
      - RUNDECK_GRAILS_URL=http://localhost:4440
      - RUNDECK_DATABASE_DRIVER=org.h2.Driver
      - RUNDECK_DATABASE_URL=jdbc:h2:file:/home/rundeck/server/data/grailsdb;MVCC=true
    ports:
      - "4440:4440"
    volumes:
      - rundeck_data:/home/rundeck/server/data
      - rundeck_projects:/home/rundeck/projects
    restart: unless-stopped

volumes:
  rundeck_data:
  rundeck_projects:
```

### Пояснения:
1. **Образ**: Используется официальный образ Rundeck с Docker Hub
2. **Порты**: Rundeck будет доступен на порту 4440
3. **Тома**: 
   - `rundeck_data` - для хранения данных Rundeck
   - `rundeck_projects` - для хранения проектов
4. **Переменные окружения**:
   - `RUNDECK_GRAILS_URL` - базовый URL для доступа к Rundeck
   - Настройки БД по умолчанию используют H2 (встроенную БД)

### Для production использования рекомендуется:
1. Заменить H2 на PostgreSQL или MySQL
2. Настроить внешний том для хранения данных
3. Добавить аутентификацию

Нужна ли вам более сложная конфигурация с внешней БД или другими настройками?




Для использования Rundeck в компании я рекомендую более продвинутую конфигурацию с внешней базой данных (PostgreSQL), отдельным томом для конфигурации, SSL и дополнительными настройками безопасности.  

### **`docker-compose.yml` для production-среды**  

```yaml
version: '3.8'

services:
  rundeck:
    image: rundeck/rundeck:4.14.0  # рекомендуется фиксировать версию
    container_name: rundeck
    hostname: rundeck.company.local  # замените на ваш домен
    environment:
      - RUNDECK_GRAILS_URL=https://rundeck.company.local  # HTTPS!
      - RUNDECK_SERVER_FORWARDED=true
      - RUNDECK_DATABASE_DRIVER=org.postgresql.Driver
      - RUNDECK_DATABASE_URL=jdbc:postgresql://postgres:5432/rundeck
      - RUNDECK_DATABASE_USERNAME=rundeck
      - RUNDECK_DATABASE_PASSWORD=your_strong_password  # замените!
      - RUNDECK_SERVER_USESSL=true
      - RUNDECK_SERVER_SECURED_URL=true
      - JAVA_OPTS=-Xmx2g -Xms1g  # настройки памяти
    volumes:
      - rundeck_data:/home/rundeck/server/data
      - rundeck_config:/home/rundeck/server/config
      - rundeck_projects:/home/rundeck/projects
      - ./ssl:/etc/rundeck/ssl  # SSL-сертификаты
    ports:
      - "4440:4440"
      - "4443:4443"  # HTTPS
    depends_on:
      - postgres
    restart: unless-stopped
    networks:
      - rundeck_net

  postgres:
    image: postgres:13
    container_name: rundeck_postgres
    environment:
      - POSTGRES_DB=rundeck
      - POSTGRES_USER=rundeck
      - POSTGRES_PASSWORD=your_strong_password  # замените!
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - rundeck_net
    restart: unless-stopped

volumes:
  rundeck_data:
  rundeck_config:
  rundeck_projects:
  pg_data:

networks:
  rundeck_net:
    driver: bridge
```

---

### **Дополнительные настройки для production**  

#### **1. SSL-сертификаты**  
Поместите сертификаты (`cert.pem`, `key.pem`) в папку `./ssl` и добавьте в `docker-compose.yml`:  
```yaml
volumes:
  - ./ssl:/etc/rundeck/ssl
```

#### **2. LDAP / Active Directory (опционально)**  
Добавьте в `rundeck-config.properties` (монтируется через `rundeck_config`):  
```properties
rundeck.security.syncLdapUser=true
rundeck.security.ldap.server=ldap://your-ldap-server
rundeck.security.ldap.bindDn=CN=service_user,OU=Users,DC=company,DC=local
rundeck.security.ldap.bindPassword=your_ldap_password
rundeck.security.ldap.userBaseDn=OU=Users,DC=company,DC=local
```

#### **3. Резервное копирование**  
- **База данных**:  
  ```bash
  docker exec rundeck_postgres pg_dump -U rundeck -d rundeck > rundeck_backup.sql
  ```
- **Проекты и конфиги**:  
  ```bash
  docker cp rundeck:/home/rundeck/server/data ./backup/
  ```

---

### **Запуск**  
1. Создайте `.env`-файл с паролями (не коммитить в Git!):  
   ```env
   POSTGRES_PASSWORD=secure_pass_123
   RUNDECK_DATABASE_PASSWORD=secure_pass_123
   ```
2. Запустите:  
   ```bash
   docker-compose up -d
   ```
3. Откройте:  
   - `https://rundeck.company.local:4443` (если настроен DNS)  
   - или `http://localhost:4440` (если тестируете локально).  

---

### **Масштабирование**  
- Для HA можно развернуть **Rundeck Enterprise** с кластеризацией.  
- Для балансировки нагрузки используйте **Nginx/Traefik**.  

Нужны ли дополнительные настройки под ваш кейс?



Вот **продвинутая production-конфигурация Rundeck** с кластеризацией (HA), внешней аутентификацией (LDAP/OAuth), мониторингом (Prometheus+Grafana), брокером сообщений (ActiveMQ) и резервным хранилищем (S3/MinIO).  

---

### **`docker-compose.ha.yml`** (High Availability)  
```yaml
version: '3.8'

services:
  # Rundeck Cluster (2 ноды)
  rundeck-node1:
    image: rundeck/rundeck:4.14.0
    hostname: rundeck-node1.company.local
    environment:
      - RUNDECK_GRAILS_URL=https://rundeck.company.local
      - RUNDECK_SERVER_FORWARDED=true
      - RUNDECK_CLUSTER_MODE=active
      - RUNDECK_DATABASE_DRIVER=org.postgresql.Driver
      - RUNDECK_DATABASE_URL=jdbc:postgresql://postgres:5432/rundeck
      - RUNDECK_DATABASE_USERNAME=rundeck
      - RUNDECK_DATABASE_PASSWORD=${DB_PASSWORD}
      - RUNDECK_SERVER_USESSL=true
      - RUNDECK_STORAGE_PROVIDER=s3
      - RUNDECK_S3_BUCKET=rundeck-backups
      - RUNDECK_S3_ENDPOINT=http://minio:9000
      - RUNDECK_S3_ACCESS_KEY=${MINIO_ACCESS_KEY}
      - RUNDECK_S3_SECRET_KEY=${MINIO_SECRET_KEY}
      - RUNDECK_QUARTZ_JOBSTORE_CLUSTERED=true
      - RUNDECK_QUARTZ_CLUSTER_MODE=active
    volumes:
      - rundeck_config:/home/rundeck/server/config
      - ./ssl:/etc/rundeck/ssl
    networks:
      - rundeck_net
    depends_on:
      - postgres
      - activemq
      - minio
    deploy:
      replicas: 1
      resources:
        limits:
          memory: 4G
          cpus: '2'

  rundeck-node2:
    image: rundeck/rundeck:4.14.0
    hostname: rundeck-node2.company.local
    environment:
      - RUNDECK_GRAILS_URL=https://rundeck.company.local
      - RUNDECK_CLUSTER_MODE=passive
      # ... (аналогично node1)
    networks:
      - rundeck_net
    depends_on:
      - rundeck-node1

  # База данных (PostgreSQL + реплика)
  postgres:
    image: postgres:13
    environment:
      - POSTGRES_DB=rundeck
      - POSTGRES_USER=rundeck
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_INITDB_ARGS=--encoding=UTF-8 --locale=C
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./postgres-backups:/backups
    deploy:
      placement:
        constraints:
          - node.role == manager
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U rundeck"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Брокер сообщений (для кластеризации)
  activemq:
    image: rmohr/activemq:5.15.9
    ports:
      - "61616:61616"  # AMQP
    volumes:
      - activemq_data:/data
    environment:
      - ACTIVEMQ_ADMIN_LOGIN=admin
      - ACTIVEMQ_ADMIN_PASSWORD=${ACTIVEMQ_PASSWORD}

  # Хранилище (MinIO вместо S3)
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data
    environment:
      - MINIO_ROOT_USER=${MINIO_ACCESS_KEY}
      - MINIO_ROOT_PASSWORD=${MINIO_SECRET_KEY}
    ports:
      - "9000:9000"  # S3 API
      - "9001:9001"  # Web UI

  # Мониторинг (Prometheus + Grafana)
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

  # Reverse Proxy (Nginx + Let's Encrypt)
  traefik:
    image: traefik:v2.5
    command:
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@company.local"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./letsencrypt:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock

volumes:
  rundeck_config:
  pg_data:
  activemq_data:
  minio_data:
  grafana_data:

networks:
  rundeck_net:
    driver: overlay
```

---

### **Ключевые особенности**  

#### **1. Высокая доступность (HA)**  
- **2 ноды Rundeck** (active-passive) с общим хранилищем (MinIO/S3).  
- **ActiveMQ** для координации задач в кластере.  
- **PostgreSQL с репликой** (можно добавить `pgpool`).  

#### **2. Безопасность**  
- **Traefik** с автоматическими SSL-сертификатами (Let's Encrypt).  
- **LDAP/OAuth** (настраивается в `rundeck-config.properties`).  
- **Secrets через `.env`** (не храним пароли в коде).  

#### **3. Мониторинг**  
- **Prometheus** + **Grafana** для метрик Rundeck (JVM, задачи, API-запросы).  
- Пример дашборда: [Grafana Dashboard for Rundeck](https://grafana.com/grafana/dashboards/18046).  

#### **4. Хранение данных**  
- **MinIO** (S3-совместимое хранилище) для бэкапов и артефактов.  
- **Ежедневные бэкапы** Postgres + MinIO.  

#### **5. Масштабируемость**  
- **Docker Swarm/Kubernetes**-ready (используется `deploy.replicas`).  
- **Отдельные сети** для изоляции сервисов.  

---

### **Деплой**  
1. Создайте `.env`-файл:  
   ```env
   DB_PASSWORD=secure_pass_123
   MINIO_ACCESS_KEY=minioadmin
   MINIO_SECRET_KEY=minioadmin123
   ACTIVEMQ_PASSWORD=activemq_pass_123
   ```
2. Запустите:  
   ```bash
   docker stack deploy -c docker-compose.ha.yml rundeck
   ```
3. Настройте:  
   - **Grafana**: импортируйте дашборд для Rundeck.  
   - **Traefik**: добавьте DNS-записи для `rundeck.company.local`.  

---

### **Что ещё можно добавить?**  
- **Terraform-модуль** для развертывания в облаке (AWS/GCP).  
- **Ansible-плейбук** для управления конфигурацией.  
- **Slack/Teams-уведомления** через Webhooks.  

Нужна ли помощь с настройкой конкретного компонента?



### **Продвинутая конфигурация Rundeck с Ansible (HA + Docker + мониторинг + резервное копирование)**  

Это решение включает:  
- **Ansible-роли** для автоматизации развертывания Rundeck в Docker (Swarm/Kubernetes)  
- **Плейбуки** для настройки кластера, LDAP, мониторинга и бэкапов  
- **Интеграцию с Terraform** (опционально) для облачного развертывания  

---

## **1. Структура проекта**  
```bash
rundeck-ansible/
├── ansible.cfg            # Конфиг Ansible
├── inventory/
│   ├── production/        # Инвентарь для prod
│   │   ├── hosts.yml      # Группы серверов
│   │   └── group_vars/    # Переменные для групп
├── roles/
│   ├── rundeck_docker/    # Роль для развертывания Rundeck
│   ├── postgres_ha/       # Настройка PostgreSQL с репликацией
│   ├── minio/             # Развертывание MinIO (S3)
│   ├── prometheus/        # Мониторинг Rundeck
│   └── traefik/           # Reverse Proxy + SSL
├── playbooks/
│   ├── deploy_rundeck.yml # Основной плейбук
│   ├── backup.yml         # Бэкапы
│   └── monitoring.yml     # Настройка Grafana
└── templates/            # Шаблоны конфигов
```

---

## **2. Роль `rundeck_docker`**  
### **`roles/rundeck_docker/tasks/main.yml`**  
```yaml
- name: "Создать сеть Docker"
  docker_network:
    name: rundeck_net
    driver: overlay
    attachable: true

- name: "Запустить PostgreSQL"
  docker_container:
    name: rundeck_postgres
    image: postgres:13
    env:
      POSTGRES_DB: "{{ rundeck_db_name }}"
      POSTGRES_USER: "{{ rundeck_db_user }}"
      POSTGRES_PASSWORD: "{{ rundeck_db_password }}"
    volumes:
      - "pg_data:/var/lib/postgresql/data"
    networks:
      - name: rundeck_net
    restart_policy: unless-stopped

- name: "Запустить Rundeck (Node 1)"
  docker_container:
    name: rundeck_node1
    image: rundeck/rundeck:4.14.0
    env:
      RUNDECK_GRAILS_URL: "https://{{ rundeck_domain }}"
      RUNDECK_DATABASE_URL: "jdbc:postgresql://rundeck_postgres:5432/{{ rundeck_db_name }}"
      RUNDECK_DATABASE_PASSWORD: "{{ rundeck_db_password }}"
    volumes:
      - "rundeck_data:/home/rundeck/server/data"
    ports:
      - "4440:4440"
    networks:
      - name: rundeck_net
    depends_on:
      - rundeck_postgres
```

---

## **3. Плейбук `deploy_rundeck.yml`**  
```yaml
- hosts: rundeck_servers
  become: true
  vars_files:
    - "inventory/production/group_vars/rundeck.yml"
  
  roles:
    - role: rundeck_docker
      tags: docker
    - role: postgres_ha
      tags: database
    - role: minio
      tags: storage
    - role: traefik
      tags: proxy
```

---

## **4. Настройка LDAP (в `group_vars/rundeck.yml`)**  
```yaml
rundeck_ldap:
  enabled: true
  server: "ldap://ldap.company.local"
  bind_dn: "CN=rundeck,OU=Service Accounts,DC=company,DC=local"
  bind_password: "{{ vault_ldap_password }}"
  user_base_dn: "OU=Users,DC=company,DC=local"
  role_mapping: |
    "admin" -> "CN=Rundeck-Admins,OU=Groups,DC=company,DC=local"
    "user" -> "CN=Rundeck-Users,OU=Groups,DC=company,DC=local"
```

---

## **5. Плейбук для бэкапов (`backup.yml`)**  
```yaml
- hosts: rundeck_servers
  tasks:
    - name: "Бэкап PostgreSQL"
      command: >
        docker exec rundeck_postgres pg_dump -U {{ rundeck_db_user }} -d {{ rundeck_db_name }} > /backups/rundeck_db_$(date +%F).sql

    - name: "Синхронизировать бэкапы в MinIO"
      command: >
        docker run --rm -v /backups:/data minio/mc mc cp /data/ s3/rundeck-backups/
      environment:
        MC_HOST_s3: "http://{{ minio_access_key }}:{{ minio_secret_key }}@minio:9000"
```

---

## **6. Запуск**  
```bash
# Развернуть Rundeck
ansible-playbook -i inventory/production/hosts.yml playbooks/deploy_rundeck.yml

# Настроить мониторинг
ansible-playbook -i inventory/production/hosts.yml playbooks/monitoring.yml
```

---

### **Дополнительные улучшения**  
1. **Vault для секретов**:  
   ```yaml
   ansible-vault encrypt inventory/production/group_vars/secrets.yml
   ```
2. **Terraform + AWS/GCP**:  
   ```hcl
   module "rundeck" {
     source  = "terraform-aws-modules/ecs/aws"
     cluster_name = "rundeck-cluster"
   }
   ```
3. **Уведомления в Slack**:  
   ```yaml
   rundeck_notifications:
     slack:
       webhook_url: "https://hooks.slack.com/services/XXX"
   ```

---

### **Итог**  
✅ **Автоматизированное развертывание** Rundeck в Docker  
✅ **High Availability** (PostgreSQL + ActiveMQ)  
✅ **Безопасность** (LDAP, Traefik + Let's Encrypt)  
✅ **Мониторинг** (Prometheus + Grafana)  
✅ **Резервное копирование** (MinIO/S3)  

Нужна помощь с конкретной частью (например, настройкой Terraform или кастомных ролей)?