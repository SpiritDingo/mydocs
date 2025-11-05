Полное корпоративное решение: Keycloak + PostgreSQL + Nginx в Docker

Вот полное решение для развертывания Keycloak в корпоративной среде с Nginx в качестве обратного прокси. Я добавил обходные пути для работы без SSL-сертификата, но с важными оговорками по безопасности.

📁 Структура проекта

```
keycloak-corporate/
├── docker-compose.yml
├── nginx/
│   └── nginx.conf
├── keycloak/
│   └── Dockerfile
└── postgres/
    └── init.sql
```

1. Docker Compose конфигурация

docker-compose.yml

```yaml
version: '3.8'

services:
  # PostgreSQL база данных
  postgres:
    image: postgres:16
    container_name: keycloak_postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-keycloak_password}
    networks:
      - keycloak_network
    restart: unless-stopped

  # Keycloak сервер
  keycloak:
    build: ./keycloak
    container_name: keycloak_server
    environment:
      KC_HOSTNAME: ${KEYCLOAK_HOST:-localhost}
      KC_HOSTNAME_PORT: 8080
      KC_HOSTNAME_STRICT: false
      KC_HOSTNAME_STRICT_HTTPS: false
      
      # Админ доступ
      KEYCLOAK_ADMIN: ${KEYCLOAK_ADMIN:-admin}
      KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD:-admin_password}
      
      # База данных
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: ${POSTGRES_PASSWORD:-keycloak_password}
      
      # Опции для работы без HTTPS
      KC_HTTP_ENABLED: true
      KC_PROXY: edge
      
      # Настройки для корпоративного использования
      KC_LOG_LEVEL: INFO
      KC_METRICS_ENABLED: true
      KC_HEALTH_ENABLED: true
    depends_on:
      - postgres
    networks:
      - keycloak_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/ready"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Nginx обратный прокси
  nginx:
    image: nginx:alpine
    container_name: keycloak_nginx
    ports:
      - "80:80"
      - "8080:80"  # Альтернативный порт
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
      - ./nginx/security.conf:/etc/nginx/security.conf
    depends_on:
      - keycloak
    networks:
      - keycloak_network
    restart: unless-stopped

volumes:
  postgres_data:

networks:
  keycloak_network:
    driver: bridge
```

2. Кастомный Dockerfile для Keycloak

keycloak/Dockerfile

```dockerfile
FROM quay.io/keycloak/keycloak:24.0.4 as builder

# Установка рабочей метки для оптимизированного образа
WORKDIR /opt/keycloak

# Настройка для production build
RUN /opt/keycloak/bin/kc.sh build --db=postgres --features=token-exchange,admin-fine-grained-authz,declarative-user-profile

FROM quay.io/keycloak/keycloak:24.0.4

# Копируем билд из предыдущего stage
COPY --from=builder /opt/keycloak/ /opt/keycloak/

# Устанавливаем провайдеры (опционально)
RUN curl -sL https://github.com/aerogear/keycloak-metrics-spi/releases/download/2.5.3/keycloak-metrics-spi-2.5.3.jar -o /opt/keycloak/providers/keycloak-metrics-spi-2.5.3.jar

# Настройка времени выполнения
ENTRYPOINT ["/opt/keycloak/bin/kc.sh"]
CMD ["start", "--optimized", "--http-enabled=true", "--hostname-strict=false", "--hostname-strict-https=false"]
```

3. Конфигурация Nginx

nginx/nginx.conf

```nginx
server {
    listen 80;
    server_name ${KEYCLOAK_HOST:-localhost};
    
    # Security headers
    include /etc/nginx/security.conf;
    
    # Проксирование на Keycloak
    location / {
        proxy_pass http://keycloak:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        
        # Таймауты
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
        
        # Буферизация
        proxy_buffering off;
        proxy_request_buffering off;
    }
    
    # Health check endpoint
    location /health {
        proxy_pass http://keycloak:8080/health;
        proxy_set_header Host $host;
        access_log off;
    }
    
    # Metrics endpoint
    location /metrics {
        proxy_pass http://keycloak:8080/metrics;
        proxy_set_header Host $host;
        access_log off;
    }
    
    # Базовые настройки
    client_max_body_size 10m;
    access_log /var/log/nginx/keycloak_access.log;
    error_log /var/log/nginx/keycloak_error.log;
}

# Резервный сервер для всех остальных хостов
server {
    listen 80 default_server;
    server_name _;
    return 444;
}
```

nginx/security.conf

```nginx
# Security headers
add_header X-Frame-Options DENY always;
add_header X-Content-Type-Options nosniff always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(),midi=(),sync-xhr=(),microphone=(),camera=(),magnetometer=(),gyroscope=(),fullscreen=(self),payment=()" always;

# Заголовки для Keycloak
add_header X-Robots-Tag "noindex, nofollow" always;

# Базовые security настройки
server_tokens off;
```

4. Инициализация базы данных

postgres/init.sql

```sql
-- Дополнительные настройки для PostgreSQL
ALTER SYSTEM SET max_connections = 200;
ALTER SYSTEM SET shared_buffers = '256MB';
ALTER SYSTEM SET effective_cache_size = '1GB';
ALTER SYSTEM SET maintenance_work_mem = '64MB';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET default_statistics_target = 100;

-- Создание дополнительных пользователей/баз при необходимости
SELECT pg_reload_conf();
```

5. Файл переменных окружения

.env

```bash
# Postgres настройки
POSTGRES_PASSWORD=secure_corporate_password_123

# Keycloak настройки
KEYCLOAK_HOST=keycloak.corporate.local
KEYCLOAK_ADMIN=corporate_admin
KEYCLOAK_ADMIN_PASSWORD=StrongAdminPassword!2024

# Прокси настройки
KC_PROXY_HEADERS=xforwarded
KC_PROXY_ADDRESS_FORWARDING=true
```

6. Скрипты для управления

start.sh

```bash
#!/bin/bash

echo "Starting Keycloak Corporate Solution..."
echo "WARNING: Running without SSL certificate - NOT recommended for production!"

# Проверка наличия docker-compose
if ! command -v docker-compose &> /dev/null; then
    echo "Error: docker-compose not found. Please install docker-compose."
    exit 1
fi

# Создание сетей и volumes
docker network create keycloak_network 2>/dev/null || true

# Запуск сервисов
docker-compose up -d

echo "Services are starting..."
echo "Keycloak will be available at: http://localhost"
echo "Keycloak Admin Console: http://localhost/admin"
echo ""
echo "To check status: docker-compose ps"
echo "To view logs: docker-compose logs -f"
```

stop.sh

```bash
#!/bin/bash

echo "Stopping Keycloak Corporate Solution..."

docker-compose down

echo "Services stopped."
```

update.sh

```bash
#!/bin/bash

echo "Updating Keycloak Corporate Solution..."

docker-compose down
docker-compose pull
docker-compose build --pull
docker-compose up -d

echo "Update completed."
```

7. Дополнительные утилиты

backup.sh

```bash
#!/bin/bash

BACKUP_DIR="./backups"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

echo "Creating backup..."
docker-compose exec postgres pg_dump -U keycloak keycloak > $BACKUP_DIR/keycloak_backup_$DATE.sql

echo "Backup created: $BACKUP_DIR/keycloak_backup_$DATE.sql"
```

🚀 Запуск решения

1. Создайте структуру директорий:

```bash
mkdir -p keycloak-corporate/{nginx,keycloak,postgres,backups}
cd keycloak-corporate
```

1. Создайте все файлы как указано выше
2. Сделайте скрипты исполняемыми:

```bash
chmod +x *.sh
```

1. Запустите систему:

```bash
./start.sh
```

1. Проверьте статус:

```bash
docker-compose ps
docker-compose logs -f keycloak
```

🔧 Доступ к системе

· Keycloak Admin Console: http://localhost/admin
· Health Checks: http://localhost/health
· Metrics: http://localhost/metrics

⚠️ ВАЖНЫЕ ПРЕДУПРЕЖДЕНИЯ ДЛЯ КОРПОРАТИВНОГО ИСПОЛЬЗОВАНИЯ

Проблемы безопасности при работе без SSL:

1. Аутентификационные данные передаются в открытом виде
2. Токены доступа могут быть перехвачены
3. Нарушение compliance требований
4. Уязвимость для MITM-атак

Временные решения:

1. Используйте только во внутренних сетях с ограниченным доступом
2. Реализуйте VPN для доступа к Keycloak
3. Настройте корпоративный firewall для ограничения доступа
4. Планируйте переход на HTTPS как можно скорее

Для production использования обязательно:

1. Получите SSL сертификат (Let's Encrypt бесплатно)
2. Обновите nginx.conf для работы на порту 443
3. Настройте автоматическое обновление сертификатов
4. Включите strict HTTPS в Keycloak

Это решение предоставляет полную функциональность Keycloak в корпоративной среде, но требует скорейшего перехода на защищенное соединение для реального использования.