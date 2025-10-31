Обновленный docker-compose.yml с PostgreSQL 17 и последней версией Keycloak:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:17-alpine
    container_name: keycloak_postgres
    hostname: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/backups:/backups
      - ./postgres/init:/docker-entrypoint-initdb.d
      - ./postgres/config:/etc/postgresql.conf
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=C --lc-collate=C --lc-ctype=C"
      POSTGRES_HOST_AUTH_METHOD: scram-sha-256
    command: 
      - "postgres"
      - "-c"
      - "config_file=/etc/postgresql.conf"
      - "-c"
      - "shared_preload_libraries=pg_stat_statements"
    networks:
      - keycloak-backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 15s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.00'
          memory: '2G'
        reservations:
          cpus: '0.50'
          memory: '1G'
    security_opt:
      - no-new-privileges:true
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  auth:
    image: quay.io/keycloak/keycloak:26.0.6
    container_name: keycloak_auth
    hostname: keycloak
    ports:
      - "${KEYCLOAK_HTTP_PORT:-8080}:8080"
      - "${KEYCLOAK_HTTPS_PORT:-8443}:8443"
    environment:
      # Базовые настройки администрирования
      KEYCLOAK_ADMIN: ${KEYCLOAK_ADMIN}
      KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
      KC_HOSTNAME_ADMIN: ${KC_HOSTNAME_ADMIN}
      KC_HOSTNAME: ${KC_HOSTNAME}
      KC_HOSTNAME_STRICT: "false"
      KC_HOSTNAME_STRICT_HTTPS: "false"
      
      # Настройки базы данных
      KC_DB: postgres
      KC_DB_PASSWORD: ${POSTGRES_PASSWORD}
      KC_DB_SCHEMA: public
      KC_DB_USERNAME: ${POSTGRES_USER}
      KC_DB_URL_HOST: postgres
      KC_DB_URL_DATABASE: ${POSTGRES_DB}
      KC_DB_URL_PORT: 5432
      KC_DB_URL_PROPERTIES: "sslmode=disable&connectTimeout=30"
      
      # Прокси и сетевые настройки
      KC_PROXY: ${KC_PROXY:-edge}
      KC_HTTP_ENABLED: "true"
      KC_HTTP_PORT: 8080
      KC_HTTPS_PORT: 8443
      
      # Безопасность и TLS
      KC_HTTPS_CERTIFICATE_FILE: /opt/keycloak/conf/tls.crt
      KC_HTTPS_CERTIFICATE_KEY_FILE: /opt/keycloak/conf/tls.key
      KC_HTTPS_PROTOCOLS: TLSv1.3,TLSv1.2
      
      # Производительность и кэширование
      KC_CACHE: local
      KC_CACHE_STACK: tcp
      KC_CACHE_CONFIG_FILE: /opt/keycloak/conf/cache-ispn.xml
      KC_HEAP_SIZE: 1024M
      KC_METRICS_ENABLED: ${KC_METRICS_ENABLED:-true}
      KC_HEALTH_ENABLED: ${KC_HEALTH_ENABLED:-true}
      
      # Логирование
      KC_LOG_LEVEL: ${KC_LOG_LEVEL:-INFO}
      KC_LOG_CONSOLE_OUTPUT: json
      KC_LOG_CONSOLE_COLOR: "false"
      
      # Функции и фичи
      KC_FEATURES: "token-exchange,admin-fine-grained-authz,declarative-user-profile,recovery-codes,web-authn"
      KC_FEATURES_DISABLED: "admin2,scripts"
      
      # Оптимизации для продакшена
      KC_TRANSACTION_XA_ENABLED: "false"
      
    depends_on:
      postgres:
        condition: service_healthy
    command:
      - start
      - --optimized
      - --spi-connections-jpa-default-migration-strategy=update
    volumes:
      - keycloak_data:/opt/keycloak/data
      - ./auth/import:/opt/keycloak/data/import
      - ./auth/themes:/opt/keycloak/themes
      - ./auth/conf:/opt/keycloak/conf
      - ./auth/providers:/opt/keycloak/providers
      - ./auth/cache:/opt/keycloak/cache
      - ./logs/keycloak:/opt/keycloak/log
    networks:
      - keycloak-backend
      - keycloak-frontend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/ready"]
      interval: 30s
      timeout: 20s
      retries: 5
      start_period: 120s
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2.00'
          memory: '4G'
        reservations:
          cpus: '1.00'
          memory: '2G'
    security_opt:
      - no-new-privileges:true
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"

  # Обратный прокси с HTTPS терминацией
  nginx:
    image: nginx:1.27-alpine
    container_name: keycloak_nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/ssl:/etc/nginx/ssl
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./logs/nginx:/var/log/nginx
    depends_on:
      - auth
    networks:
      - keycloak-frontend
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: '512M'

networks:
  keycloak-backend:
    driver: bridge
    internal: false
    ipam:
      config:
        - subnet: 172.20.0.0/24
  keycloak-frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.21.0.0/24

volumes:
  postgres_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ${PWD}/data/postgres
  keycloak_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ${PWD}/data/keycloak
```

Дополнительные конфигурационные файлы:

postgres/config/postgresql.conf:

```conf
# PostgreSQL 17 Configuration
listen_addresses = '*'
port = 5432
max_connections = 200
shared_buffers = 512MB
effective_cache_size = 1GB
work_mem = 4MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
max_wal_size = 2GB
min_wal_size = 1GB

# Logging
log_destination = 'stderr'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_file_mode = 0600
log_rotation_age = 1d
log_rotation_size = 100MB
log_min_duration_statement = 1000

# Performance
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
```

nginx/nginx.conf:

```conf
events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    include /etc/nginx/conf.d/*.conf;
}
```

nginx/conf.d/keycloak.conf:

```conf
upstream keycloak {
    server auth:8080;
}

server {
    listen 80;
    server_name ${KC_HOSTNAME};
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name ${KC_HOSTNAME};

    ssl_certificate /etc/nginx/ssl/tls.crt;
    ssl_certificate_key /etc/nginx/ssl/tls.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    client_max_body_size 100M;
    client_body_timeout 60s;
    client_header_timeout 60s;

    location / {
        proxy_pass http://keycloak;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        
        proxy_connect_timeout 30s;
        proxy_send_timeout 90s;
        proxy_read_timeout 90s;
        proxy_buffering off;
    }

    location /health {
        proxy_pass http://keycloak/health;
        access_log off;
    }

    location /metrics {
        proxy_pass http://keycloak/metrics;
        access_log off;
    }
}
```

Обновленный .env файл:

```env
# PostgreSQL 17 Configuration
POSTGRES_DB=keycloak
POSTGRES_USER=keycloak_user
POSTGRES_PASSWORD=your_secure_password_here

# Keycloak Configuration
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=your_secure_admin_password_here
KC_HOSTNAME_ADMIN=auth.yourcompany.com
KC_HOSTNAME=auth.yourcompany.com

# Network Ports
KEYCLOAK_HTTP_PORT=8080
KEYCLOAK_HTTPS_PORT=8443

# Additional Settings
KC_PROXY=edge
KC_LOG_LEVEL=INFO
KC_METRICS_ENABLED=true
KC_HEALTH_ENABLED=true
```

Основные улучшения:

1. PostgreSQL 17 с оптимизированной конфигурацией
2. Keycloak 26.0.6 (последняя стабильная версия на момент написания)
3. Улучшенная безопасность с настроенным SSL/TLS
4. Производительность - оптимизированные настройки кэширования и памяти
5. Мониторинг - расширенные health checks и метрики
6. Логирование в JSON формате для лучшей интеграции
7. Nginx как обратный прокси с HTTPS терминацией
8. Сетевые улучшения - выделенные подсети

Для запуска используйте:

```bash
mkdir -p {postgres/{backups,init,config},auth/{import,themes,conf,providers,cache},nginx/{conf.d,ssl},logs/{keycloak,nginx},data/{postgres,keycloak}}

# Генерация SSL сертификатов (для разработки)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/ssl/tls.key \
  -out nginx/ssl/tls.crt \
  -subj "/CN=auth.yourcompany.com"

docker-compose up -d
```