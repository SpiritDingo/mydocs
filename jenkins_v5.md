Полное корпоративное решение Jenkins с nginx

Структура проекта

```
jenkins-corporate/
├── .env
├── docker-compose.yml
├── jenkins/
│   └── Dockerfile
├── nginx/
│   ├── Dockerfile
│   ├── docker-entrypoint.sh
│   └── nginx.conf.template
├── scripts/
│   ├── init.sh
│   ├── setup-permissions.sh
│   ├── setup-certs.sh
│   └── install-plugins.sh
├── jenkins_home/
│   ├── casc.yaml
│   └── init.groovy.d/
└── certs/
```

1. .env файл

```bash
# Jenkins
JENKINS_VERSION=latest
JENKINS_HTTP_PORT=8080
JENKINS_HTTPS_PORT=8443

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

# Плагины
PLUGINS=ansible terraform workflow-aggregator job-dsl pipeline-groovy-lib gitlab github-branch-source powershell active-directory matrix-auth ldap email-ext credentials-binding docker-workflow ssh-slaves ws-cleanup pipeline-aws

# Volume paths
JENKINS_DATA=./jenkins_home
CERTS_DIR=./certs

# Jenkins Admin (для начальной настройки)
JENKINS_ADMIN_USER=admin
JENKINS_ADMIN_PASSWORD=changeit
```

2. docker-compose.yml

```yaml
version: '3.8'

services:
  jenkins:
    build: 
      context: ./jenkins
      dockerfile: Dockerfile
    container_name: jenkins
    restart: unless-stopped
    hostname: ${JENKINS_DOMAIN:-jenkins.company.com}
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false -Djavax.net.ssl.trustStore=/var/jenkins_certs/truststore.jks -Djavax.net.ssl.trustStorePassword=changeit
      - JENKINS_OPTS=--httpPort=8080
      - CASC_JENKINS_CONFIG=/var/jenkins_home/casc.yaml
      - JENKINS_ADMIN_ID=${JENKINS_ADMIN_USER}
      - JENKINS_ADMIN_PASSWORD=${JENKINS_ADMIN_PASSWORD}
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
    labels:
      - "description=Jenkins CI/CD Server"

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
      jenkins:
        condition: service_healthy
    networks:
      - jenkins-network
    labels:
      - "description=NGINX Reverse Proxy"

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

3. Jenkins Dockerfile

```dockerfile
FROM jenkins/jenkins:latest

USER root

# Устанавливаем необходимые пакеты
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    git \
    unzip \
    python3 \
    python3-pip \
    ansible \
    software-properties-common \
    && rm -rf /var/lib/apt/lists/*

# Устанавливаем Terraform
RUN curl -fsSL https://apt.releases.hashicorp.com/gpg | apt-key add - \
    && apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
    && apt-get update && apt-get install -y terraform

# Устанавливаем AWS CLI
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
    && unzip awscliv2.zip \
    && ./aws/install \
    && rm -rf awscliv2.zip aws

# Устанавливаем дополнительные Python пакеты
RUN pip3 install boto3 botocore

# Создаем директории и устанавливаем права
RUN mkdir -p /var/jenkins_home \
    && chown -R jenkins:jenkins /var/jenkins_home \
    && chmod -R 755 /var/jenkins_home

# Возвращаемся к пользователю jenkins
USER jenkins

# Предустановка плагинов
RUN jenkins-plugin-cli --plugins ${PLUGINS:-ansible terraform workflow-aggregator gitlab}
```

4. Nginx конфигурация

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

# Проверяем конфигурацию nginx
nginx -t

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

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    # Security headers
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin";

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

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

        # Security - block sensitive paths
        location ~* /\. {
            deny all;
            access_log off;
            log_not_found off;
        }

        location ~* /(\.git|\.env|\.htaccess) {
            deny all;
            access_log off;
            log_not_found off;
        }
    }
}
```

5. Скрипты

scripts/init.sh

```bash
#!/bin/bash

set -e

echo "==========================================="
echo "Jenkins Corporate Setup Initialization"
echo "==========================================="

# Load environment variables
if [ -f .env ]; then
    echo "Loading environment variables from .env file"
    source .env
else
    echo "Warning: .env file not found. Using defaults."
fi

# Setup permissions
echo "=== Setting up permissions ==="
./scripts/setup-permissions.sh

# Setup certificates
echo "=== Setting up certificates ==="
./scripts/setup-certs.sh

# Build custom images
echo "=== Building Docker images ==="
docker-compose build --no-cache

# Start services
echo "=== Starting services ==="
docker-compose up -d

# Wait for Jenkins to be ready
echo "=== Waiting for Jenkins to start ==="
for i in {1..30}; do
    if docker-compose logs jenkins | grep -q "Jenkins is fully up and running"; then
        echo "Jenkins is ready!"
        break
    fi
    echo "Waiting for Jenkins to start... ($i/30)"
    sleep 10
done

# Install additional plugins
echo "=== Installing additional plugins ==="
./scripts/install-plugins.sh

echo "==========================================="
echo "Initialization completed successfully!"
echo "==========================================="
echo "Jenkins URL: https://${JENKINS_DOMAIN:-jenkins.company.com}"
echo "NGINX HTTP Port: ${NGINX_HTTP_PORT:-80}"
echo "NGINX HTTPS Port: ${NGINX_HTTPS_PORT:-443}"
echo ""
echo "To get initial admin password, run:"
echo "docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword"
echo ""
echo "To view logs, run:"
echo "docker-compose logs -f"
echo "==========================================="
```

scripts/setup-permissions.sh

```bash
#!/bin/bash

set -e

echo "Setting up permissions for Jenkins..."

JENKINS_DATA_DIR=${JENKINS_DATA:-./jenkins_home}
CERTS_DIR=${CERTS_DIR:-./certs}

# Create directories
mkdir -p $JENKINS_DATA_DIR
mkdir -p $CERTS_DIR
mkdir -p nginx
mkdir -p jenkins
mkdir -p scripts

# Get current user info
CURRENT_UID=$(id -u)
CURRENT_GID=$(id -g)

echo "Current user: $USER (UID: $CURRENT_UID, GID: $CURRENT_GID)"

# Set proper ownership
echo "Setting ownership for Jenkins data directory..."
sudo chown -R $CURRENT_UID:$CURRENT_GID $JENKINS_DATA_DIR
sudo chmod -R 755 $JENKINS_DATA_DIR

echo "Setting ownership for certificates directory..."
sudo chown -R $CURRENT_UID:$CURRENT_GID $CERTS_DIR
sudo chmod -R 755 $CERTS_DIR

# Create necessary subdirectories in Jenkins home
mkdir -p $JENKINS_DATA_DIR/plugins
mkdir -p $JENKINS_DATA_DIR/jobs
mkdir -p $JENKINS_DATA_DIR/users
mkdir -p $JENKINS_DATA_DIR/secrets
mkdir -p $JENKINS_DATA_DIR/init.groovy.d
mkdir -p $JENKINS_DATA_DIR/.ssh

# Set permissions for sensitive directories
sudo chmod 700 $JENKINS_DATA_DIR/secrets
sudo chmod 700 $JENKINS_DATA_DIR/.ssh

echo "Permissions setup completed!"
```

scripts/setup-certs.sh

```bash
#!/bin/bash

set -e

echo "Setting up SSL certificates..."

JENKINS_DOMAIN=${JENKINS_DOMAIN:-jenkins.company.com}
CERTS_DIR=${CERTS_DIR:-./certs}

# Create certificates directory
mkdir -p $CERTS_DIR

# Generate self-signed certificate for nginx (replace with corporate CA in production)
echo "Generating SSL certificate for $JENKINS_DOMAIN..."
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout $CERTS_DIR/nginx.key \
    -out $CERTS_DIR/nginx.crt \
    -subj "/C=US/ST=State/L=City/O=Company/CN=$JENKINS_DOMAIN" 2>/dev/null

# Generate PKCS12 truststore for Java
echo "Generating Java truststore..."
openssl pkcs12 -export -out $CERTS_DIR/truststore.p12 \
    -inkey $CERTS_DIR/nginx.key \
    -in $CERTS_DIR/nginx.crt \
    -password pass:changeit

# Create JKS truststore (for Jenkins)
echo "Creating JKS truststore..."
keytool -import -trustcacerts -keystore $CERTS_DIR/truststore.jks \
    -storepass changeit -noprompt \
    -alias nginx -file $CERTS_DIR/nginx.crt

# Import AD certificate if exists
if [ -f "$CERTS_DIR/ad-cert.pem" ]; then
    echo "Importing AD certificate..."
    keytool -import -trustcacerts -keystore $CERTS_DIR/truststore.jks \
        -storepass changeit -noprompt -alias ad-ldaps -file $CERTS_DIR/ad-cert.pem
else
    echo "Warning: AD certificate not found at $CERTS_DIR/ad-cert.pem"
    echo "Please place your AD LDAPS certificate there and rerun this script."
    echo "You can also import it later with:"
    echo "keytool -import -trustcacerts -keystore $CERTS_DIR/truststore.jks -storepass changeit -alias ad-ldaps -file $CERTS_DIR/ad-cert.pem"
fi

# Set proper permissions for certificates
chmod 644 $CERTS_DIR/nginx.crt
chmod 600 $CERTS_DIR/nginx.key
chmod 644 $CERTS_DIR/truststore.jks

echo "Certificate setup completed!"
```

scripts/install-plugins.sh

```bash
#!/bin/bash

set -e

echo "Installing additional Jenkins plugins..."

# Wait for Jenkins to be ready
until docker-compose exec jenkins curl -s -f http://localhost:8080 > /dev/null 2>&1; do
    echo "Waiting for Jenkins to be ready..."
    sleep 10
done

# Get Jenkins CLI JAR
JENKINS_URL="http://localhost:8080"
CLI_JAR="/var/jenkins_home/jenkins-cli.jar"

# Download CLI JAR
docker-compose exec jenkins curl -s -o $CLI_JAR $JENKINS_URL/jnlpJars/jenkins-cli.jar

# Install plugins from environment variable
if [ ! -z "$PLUGINS" ]; then
    echo "Installing plugins: $PLUGINS"
    docker-compose exec jenkins java -jar $CLI_JAR -s $JENKINS_URL install-plugin $PLUGINS -deploy
fi

# Restart Jenkins to apply plugins
echo "Restarting Jenkins to apply plugins..."
docker-compose exec jenkins java -jar $CLI_JAR -s $JENKINS_URL safe-restart

echo "Plugin installation completed!"
```

6. Jenkins Configuration as Code

jenkins_home/casc.yaml

```yaml
jenkins:
  systemMessage: "Corporate Jenkins CI/CD Server"
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
        - "Job/Read:authenticated"
        - "Job/Configure:authenticated"
        - "Job/Build:authenticated"
        - "Job/Cancel:authenticated"
        - "View/Read:authenticated"
        - "Run/Replay:authenticated"
        - "Run/Update:authenticated"

  numExecutors: 5

  clouds:
    - docker:
        name: "docker"
        dockerHost:
          uri: "unix:///var/run/docker.sock"

unclassified:
  location:
    url: "https://${JENKINS_DOMAIN}"
    adminAddress: "jenkins-admins@company.com"

  mailer:
    smtpHost: "smtp.company.com"
    smtpPort: "587"
    useSsl: true
    useTls: true
    charset: "UTF-8"

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
        home: "/usr/bin/ansible"
```

7. Инструкция развертывания

Создание структуры проекта:

```bash
# Создаем основную директорию
mkdir jenkins-corporate
cd jenkins-corporate

# Создаем структуру директорий
mkdir -p nginx scripts jenkins jenkins_home/init.groovy.d certs

# Создаем все файлы как описано выше
# ... (копируем содержимое каждого файла в соответствующий файл)

# Делаем скрипты исполняемыми
chmod +x scripts/*.sh
chmod +x nginx/docker-entrypoint.sh
```

Настройка и запуск:

```bash
# 1. Настройте .env файл под вашу инфраструктуру
nano .env

# 2. Запустите полную инициализацию
./scripts/init.sh

# 3. Проверьте статус сервисов
docker-compose ps

# 4. Просмотрите логи если необходимо
docker-compose logs -f jenkins
```

Получение начального пароля:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Остановка и очистка:

```bash
# Остановка сервисов
docker-compose down

# Полная очистка (включая volumes)
docker-compose down -v
```

Особенности решения:

· Полная инфраструктура как код - все конфигурации версионируются
· Интеграция с Active Directory через LDAPS
· Поддержка всех запрошенных плагинов и инструментов
· Nginx с SSL терминацией и security headers
· Configuration as Code для Jenkins
· Автоматическая установка плагинов
· Поддержка Docker agents
· Готовность для production использования
· Обработка прав доступа для избежания permission denied ошибок

Решение полностью готово для использования в корпоративной среде после замены самоподписанных сертификатов на сертификаты корпоративного CA и настройки параметров AD под вашу инфраструктуру.