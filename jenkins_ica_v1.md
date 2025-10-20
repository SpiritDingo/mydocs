Полный корпоративный проект Jenkins с Nginx

Структура проекта

```
corporate-jenkins/
├── .env
├── docker-compose.yml
├── Dockerfile
├── jenkins.yaml
├── nginx/
│   ├── nginx.conf
│   ├── conf.d/
│   │   └── jenkins.conf
│   └── ssl/
│       ├── cert.pem
│       └── key.pem
├── scripts/
│   ├── init.sh
│   ├── generate-ssl.sh
│   └── deploy.sh
├── jenkins_data/
└── README.md
```

1. .env файл

```env
# Jenkins
JENKINS_VERSION=latest
JENKINS_HTTP_PORT=8080
JENKINS_AGENT_PORT=50000
JENKINS_ADMIN_USER=admin
JENKINS_ADMIN_PASSWORD=SecurePassword123!

# Nginx
NGINX_VERSION=latest
NGINX_HTTP_PORT=80
NGINX_HTTPS_PORT=443
SERVER_NAME=jenkins.corp.company.com

# SSL
SSL_EMAIL=admin@company.com
SSL_DOMAIN=jenkins.corp.company.com

# Active Directory
AD_DOMAIN=corp.company.com
AD_SERVER=ldap://dc.corp.company.com:389
AD_MANAGER_DN=cn=jenkins_admin,ou=ServiceAccounts,dc=corp,dc=company,dc=com
AD_MANAGER_PASSWORD=AdSecurePassword123!
AD_USER_SEARCH_BASE=ou=Users,dc=corp,dc=company,dc=com
AD_GROUP_SEARCH_BASE=ou=Groups,dc=corp,dc=company,dc=com

# Oracle Linux
ORACLE_LINUX_VERSION=8
TIMEZONE=Europe/Moscow

# Network
SUBNET=172.20.0.0/24
```

2. Dockerfile

```dockerfile
FROM jenkins/jenkins:latest

USER root

# Установка Oracle Linux репозиториев и базовых пакетов
RUN echo -e "[ol8_baseos_latest]\nname=Oracle Linux 8 BaseOS Latest\nbaseurl=https://yum.oracle.com/repo/OracleLinux/OL8/baseos/latest/x86_64/\ngpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle\ngpgcheck=1\nenabled=1" > /etc/yum.repos.d/oracle-linux.repo

# Добавление EPEL репозитория
RUN rpm -ivh https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

# Установка инструментов
RUN yum update -y && \
    yum install -y \
    python3 \
    python3-pip \
    git \
    curl \
    wget \
    unzip \
    gnupg \
    openssh-clients \
    docker-ce-cli \
    jq \
    yum-utils && \
    yum clean all

# Установка Terraform
RUN TERRAFORM_VERSION=1.5.0 && \
    wget https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip -O /tmp/terraform.zip && \
    unzip /tmp/terraform.zip -d /usr/local/bin/ && \
    rm /tmp/terraform.zip && \
    chmod +x /usr/local/bin/terraform

# Установка Ansible
RUN pip3 install ansible ansible-lint yamllint

# Установка Groovy SDK
RUN wget https://groovy.jfrog.io/artifactory/dist-release-local/groovy-zips/apache-groovy-sdk-4.0.0.zip -O /tmp/groovy.zip && \
    unzip /tmp/groovy.zip -d /opt/ && \
    ln -s /opt/groovy-4.0.0 /opt/groovy && \
    rm /tmp/groovy.zip

# Установка AWS CLI
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "/tmp/awscliv2.zip" && \
    unzip /tmp/awscliv2.zip -d /tmp && \
    /tmp/aws/install && \
    rm -rf /tmp/awscliv2.zip /tmp/aws

# Настройка PATH
ENV PATH="/opt/groovy/bin:/usr/local/aws-cli/v2/current/bin:${PATH}"

# Создание директорий для инструментов
RUN mkdir -p /var/jenkins_tools/terraform /var/jenkins_tools/ansible /var/jenkins_scripts

# Копирование скриптов
COPY scripts/ /var/jenkins_scripts/
RUN chmod +x /var/jenkins_scripts/*.sh

# Настройка времени
RUN ln -sf /usr/share/zoneinfo/${TIMEZONE} /etc/localtime

# Возврат к пользователю jenkins
USER jenkins

# Предустановка плагинов
RUN jenkins-plugin-cli --plugins \
    ansible \
    terraform \
    active-directory \
    matrix-auth \
    workflow-aggregator \
    git \
    github \
    gitlab \
    docker-workflow \
    kubernetes \
    credentials-binding \
    configuration-as-code \
    job-dsl \
    pipeline-utility-steps \
    ssh-slaves \
    email-ext \
    mailer \
    blueocean \
    aws-credentials \
    slack \
    prometheus \
    build-timeout \
    timestamper
```

3. docker-compose.yml

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:${NGINX_VERSION}-alpine
    container_name: nginx-proxy
    hostname: nginx.corp.company.com
    restart: unless-stopped
    ports:
      - "${NGINX_HTTP_PORT}:80"
      - "${NGINX_HTTPS_PORT}:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d/jenkins.conf:/etc/nginx/conf.d/jenkins.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./nginx/logs:/var/log/nginx
      - /etc/localtime:/etc/localtime:ro
    networks:
      - jenkins-network
    depends_on:
      - jenkins
    labels:
      - "description=NGINX Reverse Proxy for Jenkins"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost/nginx-health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3

  jenkins:
    build: 
      context: .
      dockerfile: Dockerfile
    container_name: jenkins-corporate
    hostname: jenkins.corp.company.com
    restart: unless-stopped
    expose:
      - "8080"
      - "50000"
    environment:
      - JAVA_OPTS=-Djava.awt.headless=true -Duser.timezone=${TIMEZONE} -Djenkins.install.runSetupWizard=false
      - JENKINS_OPTS=--argumentsRealm.roles.user=admin --argumentsRealm.passwd.admin=${JENKINS_ADMIN_PASSWORD} --argumentsRealm.roles.admin=admin --prefix=/jenkins
      - TZ=${TIMEZONE}
    volumes:
      - jenkins_data:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - ./jenkins.yaml:/var/jenkins_home/jenkins.yaml:ro
      - ./scripts:/var/jenkins_scripts:ro
      - /etc/localtime:/etc/localtime:ro
    networks:
      - jenkins-network
    labels:
      - "description=Jenkins Corporate CI/CD Server"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/jenkins/login"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

volumes:
  jenkins_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ./jenkins_data

networks:
  jenkins-network:
    driver: bridge
    ipam:
      config:
        - subnet: ${SUBNET}
```

4. Jenkins Configuration as Code (jenkins.yaml)

```yaml
jenkins:
  systemMessage: "Корпоративная система CI/CD - Oracle Linux | Active Directory Integrated"
  numExecutors: 5
  mode: NORMAL
  securityRealm:
    activeDirectory:
      domains:
        - name: "${AD_DOMAIN}"
          servers: "${AD_SERVER}"
          bindName: "${AD_MANAGER_DN}"
          bindPassword: "${AD_MANAGER_PASSWORD}"
          userSearchBase: "${AD_USER_SEARCH_BASE}"
          userSearch: "sAMAccountName={0}"
          groupSearchBase: "${AD_GROUP_SEARCH_BASE}"
          groupSearchFilter: "(& (objectClass=group) (| (cn=jenkins-*) (cn=devops*) ) )"
          removeIrrelevantGroups: true
  authorizationStrategy:
    projectMatrix:
      permissions:
        - "Overall/Administer:admin"
        - "Overall/Read:authenticated"
        - "Job/Read:authenticated"
        - "Job/Build:authenticated"
        - "Job/Cancel:authenticated"
        - "View/Read:authenticated"
        - "Credentials/View:authenticated"
  security:
    apiToken:
      creationOfLegacyTokenEnabled: false
    sshdEnabled: false
  crumbIssuer:
    standard:
      excludeClientIPFromCrumb: true
  views:
    - all:
        name: "all"
  myViewsTabBar: "standard"

unclassified:
  location:
    url: "https://${SERVER_NAME}/jenkins"
    adminAddress: "${SSL_EMAIL}"
  
  ansibleInstallation:
    installations:
      - name: "ansible-corporate"
        home: "/usr/local/bin/ansible"
  
  terraformInstallation:
    installations:
      - name: "terraform-corporate"
        home: "/usr/local/bin/terraform"
  
  globalLibraries:
    libraries:
      - name: "corporate-pipeline"
        defaultVersion: "main"
        retriever:
          modernSCM:
            scm:
              git:
                remote: "https://git.corp.company.com/infrastructure/jenkins-pipeline-libs.git"
                credentialsId: "git-corporate-credentials"

tools:
  git:
    installations:
      - name: "git-corporate"
        home: "/usr/bin/git"
  jdk:
    installations:
      - name: "corporate-jdk"
        home: "/usr/local/openjdk-11"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "git-corporate-credentials"
              username: "git-user"
              password: "git-password"
              description: "Corporate Git Credentials"
          - usernamePassword:
              scope: GLOBAL
              id: "docker-registry-corporate"
              username: "docker-user"
              password: "docker-password"
              description: "Corporate Docker Registry"
          - usernamePassword:
              scope: GLOBAL
              id: "ad-service-account"
              username: "${AD_MANAGER_DN}"
              password: "${AD_MANAGER_PASSWORD}"
              description: "Active Directory Service Account"
```

5. Nginx конфигурация (nginx/nginx.conf)

```nginx
user nginx;
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

    # Basic Settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;

    # Buffer Settings
    client_body_buffer_size 128k;
    client_max_body_size 100m;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 4k;
    output_buffers 1 32k;
    postpone_output 1460;

    # Timeout Settings
    client_body_timeout 60;
    client_header_timeout 60;
    send_timeout 60;

    # Logging Format
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" uht="$upstream_header_time" urt="$upstream_response_time"';

    access_log /var/log/nginx/access.log main;

    # Gzip Settings
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

    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    # Upstream Jenkins
    upstream jenkins {
        server jenkins:8080 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    # Include virtual hosts
    include /etc/nginx/conf.d/*.conf;
}
```

6. Виртуальный хост Jenkins (nginx/conf.d/jenkins.conf)

```nginx
# HTTP to HTTPS redirect
server {
    listen 80;
    server_name ${SERVER_NAME};
    
    # Security headers
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # Redirect to HTTPS
    return 301 https://$server_name$request_uri;
}

# HTTPS server
server {
    listen 443 ssl http2;
    server_name ${SERVER_NAME};

    # SSL Configuration
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers off;

    # SSL Security
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/nginx/ssl/cert.pem;

    # Security headers for HTTPS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';" always;

    # Proxy settings
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;

    # Timeouts
    proxy_connect_timeout 300;
    proxy_send_timeout 300;
    proxy_read_timeout 300;
    send_timeout 300;

    # Disable buffering
    proxy_buffering off;

    # Rate limiting for login
    location /jenkins/login {
        limit_req zone=api burst=5 nodelay;
        proxy_pass http://jenkins/jenkins/login;
    }

    # Jenkins location
    location /jenkins/ {
        proxy_pass http://jenkins/jenkins/;
        
        # CSRF protection
        proxy_set_header Origin "";
        proxy_hide_header X-Jenkins;
        proxy_hide_header X-Jenkins-CLI-Port;
        proxy_hide_header X-Jenkins-CLI2-Port;
        
        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Security
        proxy_cookie_path /jenkins/ /;
    }

    # Root redirect to Jenkins
    location = / {
        return 302 /jenkins/;
    }

    # Health check
    location /nginx-health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # Prometheus metrics
    location /metrics {
        access_log off;
        proxy_pass http://jenkins/jenkins/prometheus;
    }

    # Static assets cache
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        add_header Vary "Accept-Encoding";
    }

    # Deny access to sensitive files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

7. Скрипт генерации SSL (scripts/generate-ssl.sh)

```bash
#!/bin/bash

set -e

echo "Генерация SSL сертификатов..."

# Создание директорий
mkdir -p nginx/ssl

# Загрузка переменных
source .env

# Генерация самоподписанного сертификата
openssl req -x509 -nodes -days 3650 -newkey rsa:4096 \
    -keyout nginx/ssl/key.pem \
    -out nginx/ssl/cert.pem \
    -subj "/C=RU/ST=Moscow/L=Moscow/O=Company/OU=IT/CN=${SSL_DOMAIN}/emailAddress=${SSL_EMAIL}" \
    -addext "subjectAltName=DNS:${SERVER_NAME},DNS:localhost"

# Генерация DH параметров
openssl dhparam -out nginx/ssl/dhparam.pem 2048

# Установка прав
chmod 600 nginx/ssl/key.pem
chmod 644 nginx/ssl/cert.pem nginx/ssl/dhparam.pem

echo "SSL сертификаты сгенерированы:"
echo "  - nginx/ssl/cert.pem"
echo "  - nginx/ssl/key.pem" 
echo "  - nginx/ssl/dhparam.pem"
```

8. Скрипт инициализации (scripts/init.sh)

```bash
#!/bin/bash

set -e

echo "Инициализация корпоративного Jenkins..."

# Загрузка переменных
source .env

# Ожидание запуска Jenkins
echo "Ожидание запуска Jenkins..."
while ! curl -s http://localhost:8080/jenkins > /dev/null; do
    echo "Ждем запуска Jenkins..."
    sleep 15
done

echo "Jenkins запущен. Настройка корпоративных инструментов..."

# Создание скрипта настройки Groovy
cat > /tmp/configure-jenkins.groovy << 'EOF'
import jenkins.model.*
import com.cloudbees.plugins.credentials.*
import com.cloudbees.plugins.credentials.common.*
import com.cloudbees.plugins.credentials.domains.*
import com.cloudbees.plugins.credentials.impl.*
import hudson.tools.*
import hudson.util.Secret

def instance = Jenkins.getInstance()

// Настройка корневого URL Jenkins
def location = JenkinsLocationConfiguration.get()
location.setUrl("https://'${SERVER_NAME}'/jenkins/")
location.setAdminAddress("'${SSL_EMAIL}'")
location.save()

// Настройка количества исполнителей
instance.setNumExecutors(5)

// Настройка системного сообщения
instance.setSystemMessage("""
Корпоративная система CI/CD - Oracle Linux
• Active Directory Integration
• Terraform & Ansible
• Corporate Security Standards
""")

// Сохранение конфигурации
instance.save()

println "Конфигурация Jenkins завершена"
EOF

# Замена переменных в скрипте
sed -i "s|'\\${SERVER_NAME}'|${SERVER_NAME}|g" /tmp/configure-jenkins.groovy
sed -i "s|'\\${SSL_EMAIL}'|${SSL_EMAIL}|g" /tmp/configure-jenkins.groovy

# Выполнение скрипта настройки
echo "Выполнение настройки Jenkins..."
java -jar /var/jenkins_home/war/WEB-INF/jenkins-cli.jar \
    -s http://localhost:8080/jenkins \
    -auth ${JENKINS_ADMIN_USER}:${JENKINS_ADMIN_PASSWORD} \
    groovy = < /tmp/configure-jenkins.groovy

# Очистка
rm -f /tmp/configure-jenkins.groovy

echo "Корпоративный Jenkins успешно настроен!"
echo "URL: https://${SERVER_NAME}/jenkins"
echo "Admin: ${JENKINS_ADMIN_USER}"
```

9. Скрипт развертывания (scripts/deploy.sh)

```bash
#!/bin/bash

set -e

# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Функции для вывода
error() { echo -e "${RED}[ERROR]${NC} $1"; }
success() { echo -e "${GREEN}[SUCCESS]${NC} $1"; }
warning() { echo -e "${YELLOW}[WARNING]${NC} $1"; }
info() { echo -e "[INFO] $1"; }

# Проверка прав
check_permissions() {
    if [[ $EUID -eq 0 ]]; then
        error "Не запускайте скрипт от root пользователя"
        exit 1
    fi
}

# Проверка зависимостей
check_dependencies() {
    local deps=("docker" "docker-compose" "curl" "openssl")
    for dep in "${deps[@]}"; do
        if ! command -v "$dep" &> /dev/null; then
            error "Не найдена зависимость: $dep"
            exit 1
        fi
    done
    success "Все зависимости установлены"
}

# Создание структуры папок
create_structure() {
    info "Создание структуры папок..."
    
    mkdir -p \
        nginx/conf.d \
        nginx/ssl \
        nginx/logs \
        jenkins_data \
        scripts \
        backup
    
    chmod 755 nginx/conf.d nginx/ssl nginx/logs scripts backup
    chmod 750 jenkins_data
    
    success "Структура папок создана"
}

# Генерация SSL сертификатов
generate_ssl() {
    if [[ -f "nginx/ssl/cert.pem" && -f "nginx/ssl/key.pem" ]]; then
        warning "SSL сертификаты уже существуют, пропускаем генерацию"
        return 0
    fi
    
    info "Генерация SSL сертификатов..."
    chmod +x scripts/generate-ssl.sh
    ./scripts/generate-ssl.sh
    success "SSL сертификаты сгенерированы"
}

# Настройка прав
setup_permissions() {
    info "Настройка прав доступа..."
    
    chmod +x scripts/*.sh
    chown -R 1000:1000 jenkins_data
    chmod 755 jenkins_data
    
    # Права для nginx логов
    chmod 755 nginx/logs
    
    success "Права доступа настроены"
}

# Запуск сервисов
start_services() {
    info "Запуск сервисов..."
    
    docker-compose up -d
    
    # Проверка запуска
    local timeout=180
    local counter=0
    
    while ! docker-compose ps | grep -q "Up"; do
        if [ $counter -eq $timeout ]; then
            error "Таймаут запуска сервисов"
            docker-compose logs
            exit 1
        fi
        sleep 5
        counter=$((counter + 5))
    done
    
    success "Сервисы запущены"
}

# Инициализация Jenkins
init_jenkins() {
    info "Инициализация Jenkins..."
    
    # Ждем готовности Jenkins
    info "Ожидание готовности Jenkins..."
    local timeout=300
    local counter=0
    
    while ! curl -s -f "http://localhost:8080/jenkins" > /dev/null; do
        if [ $counter -eq $timeout ]; then
            error "Таймаут инициализации Jenkins"
            docker-compose logs jenkins
            exit 1
        fi
        sleep 10
        counter=$((counter + 10))
        info "Ждем... ($counter сек.)"
    done
    
    # Запуск скрипта инициализации
    docker exec -u jenkins jenkins-corporate /var/jenkins_scripts/init.sh
    
    success "Jenkins инициализирован"
}

# Проверка здоровья
health_check() {
    info "Проверка здоровья сервисов..."
    
    # Проверка nginx
    if curl -s -f "http://localhost:${NGINX_HTTP_PORT}/nginx-health" > /dev/null; then
        success "Nginx здоров"
    else
        error "Nginx не отвечает"
    fi
    
    # Проверка Jenkins
    if curl -s -f "https://${SERVER_NAME}/jenkins" > /dev/null; then
        success "Jenkins доступен"
    else
        error "Jenkins не доступен"
    fi
}

# Основная функция
main() {
    info "Начало развертывания корпоративного Jenkins"
    
    check_permissions
    check_dependencies
    
    source .env
    
    create_structure
    generate_ssl
    setup_permissions
    start_services
    init_jenkins
    health_check
    
    success "Развертывание завершено!"
    echo ""
    echo "=== ДОСТУП К СИСТЕМЕ ==="
    echo "Jenkins URL: https://${SERVER_NAME}/jenkins"
    echo "Admin пользователь: ${JENKINS_ADMIN_USER}"
    echo "Admin пароль: ${JENKINS_ADMIN_PASSWORD}"
    echo ""
    echo "=== КОМАНДЫ УПРАВЛЕНИЯ ==="
    echo "Просмотр логов: docker-compose logs -f"
    echo "Остановка: docker-compose down"
    echo "Перезапуск: docker-compose restart"
    echo ""
    echo "=== МОНИТОРИНГ ==="
    echo "Health check: http://localhost:${NGINX_HTTP_PORT}/nginx-health"
    echo ""
}

# Обработка сигналов
trap "error 'Прервано пользователем'; exit 1" SIGINT SIGTERM

# Запуск
main "$@"
```

10. README.md

```markdown
# Корпоративный Jenkins с Oracle Linux

Полная инфраструктура как код для корпоративного Jenkins сервера с интеграцией Active Directory.

## Особенности

- 🐳 **Docker Compose** - Оркестрация контейнеров
- 🔒 **Nginx + SSL** - Reverse proxy с HTTPS
- 👥 **Active Directory** - Корпоративная аутентификация
- 🛠 **Инструменты IaC** - Terraform, Ansible, Groovy, Bash
- 🐧 **Oracle Linux** - Корпоративный дистрибутив
- 📊 **Мониторинг** - Health checks, логирование
- 🔐 **Безопасность** - Security headers, rate limiting

## Быстрый старт

```bash
# Клонирование проекта
git clone <repository>
cd corporate-jenkins

# Настройка переменных (при необходимости)
cp .env.example .env
nano .env

# Запуск развертывания
chmod +x scripts/deploy.sh
./scripts/deploy.sh
```

Структура проекта

```
corporate-jenkins/
├── .env                    # Переменные окружения
├── docker-compose.yml      # Оркестрация сервисов
├── Dockerfile             # Образ Jenkins с инструментами
├── jenkins.yaml           # Конфигурация как код
├── nginx/                 # Конфигурация Nginx
├── scripts/               # Скрипты развертывания
├── jenkins_data/          # Данные Jenkins
└── README.md
```

Переменные окружения (.env)

Основные настройки в файле .env:

· JENKINS_ADMIN_PASSWORD - Пароль администратора
· SERVER_NAME - Доменное имя сервера
· AD_DOMAIN - Домен Active Directory
· AD_SERVER - Сервер LDAP
· AD_MANAGER_DN - Service account для AD

Управление сервисами

```bash
# Запуск
docker-compose up -d

# Остановка
docker-compose down

# Просмотр логов
docker-compose logs -f

# Перезапуск
docker-compose restart

# Статус сервисов
docker-compose ps
```

Безопасность

· HTTPS с современными шифрами
· Security headers (CSP, HSTS, XSS Protection)
· Rate limiting для аутентификации
· Интеграция с Active Directory
· Регулярное обновление образов

Мониторинг

· Health checks: http://localhost/nginx-health
· Логи: docker-compose logs -f nginx jenkins
· Метрики: /metrics endpoint

Поддержка

Для вопросов и поддержки обращайтесь в отдел DevOps.

```

## Запуск проекта

```bash
# 1. Скачайте все файлы в папку corporate-jenkins

# 2. Настройте переменные в .env
cp .env .env.backup
nano .env  # Отредактируйте под вашу инфраструктуру

# 3. Запустите развертывание
chmod +x scripts/deploy.sh
./scripts/deploy.sh

# 4. Проверьте доступность
curl -k https://jenkins.corp.company.com/jenkins
```

Это полное корпоративное решение готово к использованию в production среде с всеми необходимыми инструментами и настройками безопасности.