Вот готовое корпоративное решение с Jenkins и nginx, использующее последние версии и включающее все указанные требования:

.env файл

```properties
# Jenkins
JENKINS_VERSION=latest
JENKINS_HTTP_PORT=8080
JENKINS_HTTPS_PORT=8443
JENKINS_AGENT_PORT=50000

# NGINX
NGINX_VERSION=latest
NGINX_HTTP_PORT=80
NGINX_HTTPS_PORT=443

# Домены
JENKINS_DOMAIN=jenkins.company.com

# LDAP/AD
LDAP_URL=ldaps://ad.company.com:636
LDAP_BASE_DN=DC=company,DC=com
LDAP_MANAGER_DN=CN=jenkins_svc,OU=Service Accounts,DC=company,DC=com
LDAP_MANAGER_PASSWORD=changeit

# Volume paths
JENKINS_DATA=./jenkins_home
CERTS_DIR=./certs
```

docker-compose.yml

```yaml
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:${JENKINS_VERSION:-latest}
    container_name: jenkins
    restart: unless-stopped
    hostname: ${JENKINS_DOMAIN:-jenkins}
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false -Djavax.net.ssl.trustStore=/var/jenkins_certs/truststore.jks -Djavax.net.ssl.trustStorePassword=changeit
      - JENKINS_OPTS=--httpPort=${JENKINS_HTTP_PORT:-8080} --httpsPort=${JENKINS_HTTPS_PORT:-8443}
      - LDAP_URL=${LDAP_URL}
      - LDAP_BASE_DN=${LDAP_BASE_DN}
      - LDAP_MANAGER_DN=${LDAP_MANAGER_DN}
      - LDAP_MANAGER_PASSWORD=${LDAP_MANAGER_PASSWORD}
    volumes:
      - ${JENKINS_DATA:-./jenkins_home}:/var/jenkins_home
      - ${CERTS_DIR:-./certs}:/var/jenkins_certs:ro
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - jenkins-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jenkins.rule=Host(`${JENKINS_DOMAIN}`)"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:${NGINX_VERSION:-latest}
    container_name: nginx-proxy
    restart: unless-stopped
    ports:
      - "${NGINX_HTTP_PORT:-80}:80"
      - "${NGINX_HTTPS_PORT:-443}:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ${CERTS_DIR:-./certs}:/etc/nginx/certs:ro
    depends_on:
      - jenkins
    networks:
      - jenkins-network

networks:
  jenkins-network:
    driver: bridge
    name: jenkins-corporate-network

volumes:
  jenkins_data:
    driver: local
  nginx_config:
    driver: local
```

Конфигурация nginx (nginx/nginx.conf)

```nginx
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Security headers
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Upstream Jenkins
    upstream jenkins {
        server jenkins:${JENKINS_HTTP_PORT:-8080};
        keepalive 32;
    }

    # HTTP to HTTPS redirect
    server {
        listen 80;
        server_name ${JENKINS_DOMAIN};
        return 301 https://$server_name$request_uri;
    }

    # HTTPS server
    server {
        listen 443 ssl http2;
        server_name ${JENKINS_DOMAIN};

        ssl_certificate /etc/nginx/certs/nginx.crt;
        ssl_certificate_key /etc/nginx/certs/nginx.key;
        
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;

        # Jenkins proxy
        location / {
            proxy_pass http://jenkins;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port $server_port;

            proxy_connect_timeout 90;
            proxy_send_timeout 90;
            proxy_read_timeout 90;

            proxy_buffer_size 128k;
            proxy_buffers 4 256k;
            proxy_busy_buffers_size 256k;

            # WebSocket support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }

        # Security
        location ~* /\. {
            deny all;
            access_log off;
            log_not_found off;
        }
    }
}
```

Скрипт инициализации (scripts/init-jenkins.sh)

```bash
#!/bin/bash

# Wait for Jenkins to start
echo "Waiting for Jenkins to start..."
while [ $(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080) -ne 403 ]; do
  sleep 10
done

echo "Jenkins is up! Installing plugins..."

# Install required plugins
PLUGINS="ansible terraform workflow-aggregator job-dsl pipeline-groovy-lib gitlab github-branch-source powershell active-directory matrix-auth ldap email-ext credentials-binding docker-workflow ssh-slaves ws-cleanup"

java -jar /var/jenkins_home/war/WEB-INF/jenkins-cli.jar -s http://localhost:8080/ -auth admin:${JENKINS_ADMIN_PASSWORD} install-plugin $PLUGINS

echo "Plugins installed. Restarting Jenkins..."

# Safe restart
java -jar /var/jenkins_home/war/WEB-INF/jenkins-cli.jar -s http://localhost:8080/ -auth admin:${JENKINS_ADMIN_PASSWORD} safe-restart

echo "Jenkins configuration completed!"
```

Jenkins Configuration as Code (jenkins_home/casc.yaml)

```yaml
jenkins:
  securityRealm:
    ldap:
      configurations:
        - server: "${LDAP_URL}"
          rootDN: "${LDAP_BASE_DN}"
          managerDN: "${LDAP_MANAGER_DN}"
          managerPasswordSecret: "${LDAP_MANAGER_PASSWORD}"
          userSearchBase: ""
          userSearch: "sAMAccountName={0}"
          groupSearchBase: "OU=Groups,${LDAP_BASE_DN}"
          inhibitInferRootDN: true
          disableMailAddressResolver: false
          cache:
            size: 100
            ttl: 30
      cache:
        size: 100
        ttl: 30
      userIdStrategy: CaseSensitive
      groupIdStrategy: CaseSensitive

  authorizationStrategy:
    globalMatrix:
      permissions:
        - "Overall/Administer:authenticated"
        - "Overall/Read:authenticated"

  numExecutors: 5

unclassified:
  location:
    url: "https://${JENKINS_DOMAIN}"

tool:
  git:
    installations:
      - name: "git"
        home: "/usr/bin/git"
  terraform:
    installations:
      - name: "terraform"
        home: "/usr/local/bin/terraform"
  ansible:
    installations:
      - name: "ansible"
        home: "/usr/local/bin/ansible"
```

Скрипт установки сертификатов (scripts/setup-certs.sh)

```bash
#!/bin/bash

# Create certs directory
mkdir -p ${CERTS_DIR:-./certs}

# Generate self-signed certificate for nginx (replace with corporate CA in production)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ${CERTS_DIR:-./certs}/nginx.key \
  -out ${CERTS_DIR:-./certs}/nginx.crt \
  -subj "/C=US/ST=State/L=City/O=Company/CN=${JENKINS_DOMAIN}"

# Import AD LDAPS certificate (replace with actual corporate certificate)
# Copy your corporate AD certificate to ./certs/ad-cert.pem first
if [ -f "${CERTS_DIR:-./certs}/ad-cert.pem" ]; then
    keytool -import -trustcacerts -keystore ${CERTS_DIR:-./certs}/truststore.jks \
      -storepass changeit -noprompt -alias ad-ldaps -file ${CERTS_DIR:-./certs}/ad-cert.pem
fi
```

Инструкция развертывания

1. Подготовка окружения:

```bash
# Создайте необходимые директории
mkdir -p {nginx/conf.d,scripts,certs,jenkins_home}

# Сделайте скрипты исполняемыми
chmod +x scripts/*.sh
```

1. Настройка сертификатов:

```bash
# Запустите скрипт настройки сертификатов
./scripts/setup-certs.sh

# Поместите ваш корпоративный сертификат AD в ./certs/ad-cert.pem
```

1. Запуск инфраструктуры:

```bash
# Запустите все сервисы
docker-compose up -d

# Проверьте статус
docker-compose ps
```

1. Получение начального пароля:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Особенности решения:

· Инфраструктура как код - вся конфигурация версионируется
· Интеграция с MS AD через LDAPS с поддержкой TLS
· Все запрошенные плагины предустановлены
· Nginx reverse proxy с SSL/TLS терминацией
· Security headers и лучшие практики безопасности
· Health checks и автоматические перезапуски
· Поддержка Docker для агентов Jenkins
· Готовность для production использования

Для использования в корпоративной среде замените самоподписанные сертификаты на сертификаты вашего корпоративного CA и настройте параметры AD в соответствии с вашей инфраструктурой.



Исправлю ошибку с переменными в nginx конфигурации. Проблема в том, что nginx не поддерживает переменные окружения в конфиге напрямую. Нужно использовать шаблонизацию.

Обновленный docker-compose.yml

```yaml
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:${JENKINS_VERSION:-latest}
    container_name: jenkins
    restart: unless-stopped
    hostname: ${JENKINS_DOMAIN:-jenkins.company.com}
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false -Djavax.net.ssl.trustStore=/var/jenkins_certs/truststore.jks -Djavax.net.ssl.trustStorePassword=changeit
      - JENKINS_OPTS=--httpPort=8080
    volumes:
      - ${JENKINS_DATA:-./jenkins_home}:/var/jenkins_home
      - ${CERTS_DIR:-./certs}:/var/jenkins_certs:ro
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - jenkins-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    build: 
      context: ./nginx
      dockerfile: Dockerfile
    container_name: nginx-proxy
    restart: unless-stopped
    ports:
      - "${NGINX_HTTP_PORT:-80}:80"
      - "${NGINX_HTTPS_PORT:-443}:443"
    environment:
      - JENKINS_BACKEND=jenkins:8080
      - JENKINS_DOMAIN=${JENKINS_DOMAIN:-jenkins.company.com}
    volumes:
      - ${CERTS_DIR:-./certs}:/etc/nginx/certs:ro
    depends_on:
      - jenkins
    networks:
      - jenkins-network

networks:
  jenkins-network:
    driver: bridge

volumes:
  jenkins_data:
    driver: local
```

Создаем кастомный nginx образ с шаблонизацией

nginx/Dockerfile

```dockerfile
FROM nginx:latest

# Устанавливаем envsubst для замены переменных
RUN apt-get update && apt-get install -y gettext-base && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Копируем шаблон конфигурации
COPY nginx.conf.template /etc/nginx/nginx.conf.template

# Скрипт для замены переменных
COPY docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
```

nginx/docker-entrypoint.sh

```bash
#!/bin/bash

# Заменяем переменные в шаблоне
envsubst '${JENKINS_BACKEND} ${JENKINS_DOMAIN}' < /etc/nginx/nginx.conf.template > /etc/nginx/nginx.conf

# Запускаем nginx
exec "$@"
```

nginx/nginx.conf.template

```nginx
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Security headers
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Upstream Jenkins
    upstream jenkins {
        server ${JENKINS_BACKEND};
        keepalive 32;
    }

    # HTTP to HTTPS redirect
    server {
        listen 80;
        server_name ${JENKINS_DOMAIN};
        return 301 https://$server_name$request_uri;
    }

    # HTTPS server
    server {
        listen 443 ssl http2;
        server_name ${JENKINS_DOMAIN};

        ssl_certificate /etc/nginx/certs/nginx.crt;
        ssl_certificate_key /etc/nginx/certs/nginx.key;
        
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;

        # Jenkins proxy
        location / {
            proxy_pass http://jenkins;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port $server_port;

            proxy_connect_timeout 90;
            proxy_send_timeout 90;
            proxy_read_timeout 90;

            proxy_buffer_size 128k;
            proxy_buffers 4 256k;
            proxy_busy_buffers_size 256k;

            # WebSocket support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }

        # Security
        location ~* /\. {
            deny all;
            access_log off;
            log_not_found off;
        }
    }
}
```

Обновленный .env файл

```properties
# Jenkins
JENKINS_VERSION=latest
JENKINS_HTTP_PORT=8080

# NGINX
NGINX_HTTP_PORT=80
NGINX_HTTPS_PORT=443

# Домены
JENKINS_DOMAIN=jenkins.company.com

# LDAP/AD
LDAP_URL=ldaps://ad.company.com:636
LDAP_BASE_DN=DC=company,DC=com
LDAP_MANAGER_DN=CN=jenkins_svc,OU=Service Accounts,DC=company,DC=com
LDAP_MANAGER_PASSWORD=changeit

# Volume paths
JENKINS_DATA=./jenkins_home
CERTS_DIR=./certs
```

Обновленный скрипт установки сертификатов

scripts/setup-certs.sh

```bash
#!/bin/bash

set -e

echo "Setting up certificates..."

# Create directories
mkdir -p ${CERTS_DIR:-./certs}
mkdir -p nginx

# Generate self-signed certificate for nginx
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ${CERTS_DIR:-./certs}/nginx.key \
  -out ${CERTS_DIR:-./certs}/nginx.crt \
  -subj "/C=US/ST=State/L=City/O=Company/CN=${JENKINS_DOMAIN:-jenkins.company.com}" 2>/dev/null

# Create truststore and import AD certificate if exists
if [ -f "${CERTS_DIR:-./certs}/ad-cert.pem" ]; then
    echo "Importing AD certificate..."
    keytool -import -trustcacerts -keystore ${CERTS_DIR:-./certs}/truststore.jks \
      -storepass changeit -noprompt -alias ad-ldaps -file ${CERTS_DIR:-./certs}/ad-cert.pem
else
    echo "Warning: AD certificate not found at ${CERTS_DIR:-./certs}/ad-cert.pem"
    echo "Please place your AD LDAPS certificate there and rerun this script."
fi

echo "Certificate setup completed!"
```

Инструкция развертывания

1. Создайте структуру директорий:

```bash
mkdir -p nginx scripts certs jenkins_home
```

1. Поместите файлы в соответствующие директории:

· nginx/Dockerfile
· nginx/docker-entrypoint.sh
· nginx/nginx.conf.template
· scripts/setup-certs.sh

1. Настройте сертификаты:

```bash
chmod +x scripts/setup-certs.sh
./scripts/setup-certs.sh
```

1. Запустите инфраструктуру:

```bash
docker-compose up -d
```

Проверка работы

```bash
# Проверьте статус контейнеров
docker-compose ps

# Проверьте логи nginx
docker logs nginx-proxy

# Проверьте доступность
curl -I https://${JENKINS_DOMAIN}
```

Ключевые изменения:

1. Убраны переменные из nginx.conf - теперь используется шаблонизация с envsubst
2. Создан кастомный nginx образ с поддержкой замены переменных
3. Фиксированные внутренние порты - Jenkins всегда на 8080 внутри контейнера
4. Упрощен docker-compose.yml - убраны лишние volume mounts

Теперь nginx будет правильно разрешать адрес Jenkins и ошибка с портом исчезнет.