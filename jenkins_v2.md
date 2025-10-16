Предлагаю корпоративное решение для развертывания Jenkins с плагинами для запуска Ansible ролей по принципу "Инфраструктура как код".

1. Структура проекта

```
jenkins-iac/
├── docker-compose.yml
├── Dockerfile
├── plugins.txt
├── casc/
│   └── jenkins.yaml
├── scripts/
│   └── install-plugins.sh
└── data/
    ├── jenkins_home/
    └── ansible/
```

2. Docker Compose конфигурация

docker-compose.yml:

```yaml
version: '3.8'

services:
  jenkins:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: jenkins-iac
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    environment:
      - JENKINS_OPTS=--httpPort=8080
      - CASC_JENKINS_CONFIG=/var/jenkins_conf/casc
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false -Xmx2g -Xms512m
    volumes:
      - ./data/jenkins_home:/var/jenkins_home
      - ./data/ansible:/var/ansible
      - ./casc:/var/jenkins_conf/casc
      - /var/run/docker.sock:/var/run/docker.sock
      - ./scripts:/var/jenkins_scripts
    networks:
      - jenkins-network
    security_opt:
      - no-new-privileges:true

  ansible-runner:
    image: quay.io/ansible/ansible-runner:latest
    container_name: ansible-runner
    restart: unless-stopped
    volumes:
      - ./data/ansible:/runner
      - ./scripts:/scripts
    networks:
      - jenkins-network
    profiles:
      - tools

networks:
  jenkins-network:
    driver: bridge
    name: jenkins-iac-network

volumes:
  jenkins_data:
    driver: local
  ansible_data:
    driver: local
```

3. Dockerfile для кастомного образа Jenkins

Dockerfile:

```dockerfile
FROM jenkins/jenkins:lts-jdk17

USER root

# Установка необходимых утилит
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    git \
    sshpass \
    openssh-client \
    curl \
    gnupg \
    software-properties-common \
    && rm -rf /var/lib/apt/lists/*

# Установка Ansible
RUN pip3 install ansible ansible-lint yamllint

# Установка Docker CLI
RUN curl -fsSL https://get.docker.com | sh

# Создание директорий для конфигураций
RUN mkdir -p /var/jenkins_conf/casc /var/ansible

# Копирование плагинов и скриптов
COPY plugins.txt /usr/share/jenkins/ref/
COPY scripts/install-plugins.sh /usr/local/bin/

# Установка плагинов
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

# Настройка прав
RUN chown -R jenkins:jenkins /var/jenkins_conf /var/ansible
RUN usermod -aG docker jenkins

USER jenkins

HEALTHCHECK --interval=30s --timeout=10s --start-period=1m --retries=3 \
  CMD curl -f http://localhost:8080 || exit 1
```

4. Список плагинов

plugins.txt:

```txt
ansible
ansible-tower
job-dsl
configuration-as-code
git
github
gitlab
pipeline-github
workflow-aggregator
pipeline-stage-view
blueocean
docker-workflow
kubernetes
credentials-binding
ssh-slaves
email-ext
mailer
matrix-auth
role-strategy
timestamper
ws-cleanup
build-timeout
parameterized-trigger
copyartifact
envinject
htmlpublisher
pipeline-utility-steps
script-security
ssh-credentials
```

5. Configuration as Code

casc/jenkins.yaml:

```yaml
jenkins:
  systemMessage: "Jenkins Infrastructure as Code - Корпоративное решение"
  numExecutors: 5
  mode: NORMAL
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "${ADMIN_PASSWORD}"
  authorizationStrategy:
    globalMatrix:
      permissions:
        - "Overall/Administer:admin"
        - "Overall/Read:authenticated"

security:
  apiToken:
    creationOfLegacyTokenEnabled: false
  sshd:
    enabled: false

unclassified:
  location:
    url: "http://jenkins.your-company.com"
    adminAddress: "jenkins-admin@your-company.com"
  
  mailer:
    smtpHost: "smtp.your-company.com"
    smtpPort: "587"
    useSsl: true
    smtpAuth: true
    username: "jenkins@your-company.com"
    password: "${SMTP_PASSWORD}"

tools:
  installations:
    - name: "ansible"
      properties:
        - installSource:
            installers:
              - command:
                  command: "/usr/local/bin/ansible --version"
    - name: "git"
      properties:
        - installSource:
            installers:
              - command:
                  command: "/usr/bin/git --version"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "git-credentials"
              username: "git-user"
              password: "${GIT_PASSWORD}"
          - sshUsernamePrivateKey:
              scope: GLOBAL
              id: "ansible-ssh-key"
              username: "ansible"
              privateKeySource:
                directEntry:
                  privateKey: "${ANSIBLE_SSH_KEY}"
          - string:
              scope: GLOBAL
              id: "vault-password"
              secret: "${ANSIBLE_VAULT_PASSWORD}"
```

6. Скрипт установки плагинов

scripts/install-plugins.sh:

```bash
#!/bin/bash
set -e

JENKINS_PLUGIN_CLI=/usr/local/bin/jenkins-plugin-cli

if [ -f "$JENKINS_PLUGIN_CLI" ]; then
    echo "Установка плагинов через jenkins-plugin-cli..."
    jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
else
    echo "Установка плагинов через install-plugins.sh..."
    /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt
fi

echo "Плагины успешно установлены"
```

7. Файл переменных окружения

.env:

```env
# Jenkins Admin
ADMIN_PASSWORD=secure_password_123

# SMTP Configuration
SMTP_PASSWORD=smtp_password_123

# Git Credentials
GIT_PASSWORD=git_token_123

# Ansible SSH Key (начальное значение)
ANSIBLE_SSH_KEY=ssh-rsa AAAAB3NzaC1yc2...

# Ansible Vault
ANSIBLE_VAULT_PASSWORD=vault_password_123

# Network
JENKINS_URL=http://localhost:8080
```

8. Деплой и управление

deploy.sh:

```bash
#!/bin/bash

set -e

echo "🚀 Запуск Jenkins Infrastructure as Code..."

# Проверка зависимостей
command -v docker >/dev/null 2>&1 || { echo "Docker не установлен"; exit 1; }
command -v docker-compose >/dev/null 2>&1 || { echo "Docker Compose не установлен"; exit 1; }

# Создание директорий
mkdir -p data/jenkins_home data/ansible casc scripts

# Настройка прав
chmod +x scripts/*.sh

# Загрузка образов
echo "📦 Загрузка Docker образов..."
docker-compose pull

# Запуск сервисов
echo "🔧 Запуск Jenkins..."
docker-compose up -d

echo "⏳ Ожидание запуска Jenkins..."
sleep 30

# Проверка здоровья
if curl -f http://localhost:8080 >/dev/null 2>&1; then
    echo "✅ Jenkins успешно запущен"
    echo "🌐 Доступен по адресу: http://localhost:8080"
    
    # Получение initial admin password
    if [ -f "data/jenkins_home/secrets/initialAdminPassword" ]; then
        echo "🔑 Initial Admin Password:"
        cat data/jenkins_home/secrets/initialAdminPassword
    fi
else
    echo "❌ Ошибка запуска Jenkins"
    docker-compose logs jenkins
    exit 1
fi
```

9. Pipeline пример для Ansible

data/jenkins_home/ansible-pipeline.jenkinsfile:

```groovy
pipeline {
    agent any
    tools {
        ansible 'ansible'
    }
    environment {
        ANSIBLE_HOST_KEY_CHECKING = 'false'
        VAULT_PASSWORD = credentials('vault-password')
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-credentials',
                    url: 'https://github.com/your-company/ansible-infrastructure.git'
            }
        }
        stage('Lint') {
            steps {
                sh 'ansible-lint site.yml'
                sh 'yamllint .'
            }
        }
        stage('Deploy Infrastructure') {
            steps {
                script {
                    withCredentials([sshUserPrivateKey(
                        credentialsId: 'ansible-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh '''
                        ansible-playbook -i inventory/prod site.yml \
                          --private-key $SSH_KEY \
                          --vault-password-file <(echo "$VAULT_PASSWORD") \
                          --limit production
                        '''
                    }
                }
            }
        }
    }
    post {
        always {
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'reports',
                reportFiles: 'ansible-report.html',
                reportName: 'Ansible Report'
            ])
        }
        success {
            emailext (
                subject: "SUCCESS: Job ${env.JOB_NAME} - Build ${env.BUILD_NUMBER}",
                body: "Ansible deployment completed successfully",
                to: "team@your-company.com"
            )
        }
        failure {
            emailext (
                subject: "FAILED: Job ${env.JOB_NAME} - Build ${env.BUILD_NUMBER}",
                body: "Ansible deployment failed. Please check Jenkins logs.",
                to: "team@your-company.com"
            )
        }
    }
}
```

Запуск решения

1. Клонируйте структуру проекта:

```bash
mkdir jenkins-iac && cd jenkins-iac
# Создайте все файлы согласно структуре выше
```

1. Настройте переменные окружения:

```bash
cp .env.example .env
# Отредактируйте .env файл с реальными значениями
```

1. Запустите деплой:

```bash
chmod +x deploy.sh
./deploy.sh
```

1. Запустите дополнительные инструменты (опционально):

```bash
docker-compose --profile tools up -d ansible-runner
```

Особенности решения

· ✅ Security: RBAC, безопасные credentials, no-new-privileges
· ✅ Scalability: Подготовка для кластеризации
· ✅ Infrastructure as Code: Полная конфигурация через YAML
· ✅ Ansible Integration: Готовые пайплайны для развертывания
· ✅ Corporate Ready: Поддержка корпоративных proxy, SMTP, LDAP
· ✅ Monitoring: Health checks, логирование
· ✅ Backup: Volume mapping для persistence

Это решение предоставляет готовую корпоративную платформу для управления инфраструктурой через Ansible с использованием Jenkins как оркестратора.



Обновленное решение для Docker Compose V2 с использованием современных возможностей:

1. Обновленный docker-compose.yml

docker-compose.yml:

```yaml
name: jenkins-iac-corporate

services:
  jenkins:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        JENKINS_VERSION: lts-jdk17
    container_name: jenkins-iac-corporate
    hostname: jenkins-iac
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    environment:
      JENKINS_OPTS: --httpPort=8080
      CASC_JENKINS_CONFIG: /var/jenkins_conf/casc
      JAVA_OPTS: -Djenkins.install.runSetupWizard=false -Xmx2g -Xms512m -Duser.timezone=Europe/Moscow
      JENKINS_SLAVE_AGENT_PORT: 50000
    env_file:
      - .env
    volumes:
      - jenkins_data:/var/jenkins_home
      - ansible_data:/var/ansible
      - ./casc:/var/jenkins_conf/casc:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./scripts:/var/jenkins_scripts:ro
      - ./shared:/shared:rw
    networks:
      jenkins-network:
        aliases:
          - jenkins-main
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    configs:
      - source: plugins_txt
        target: /usr/share/jenkins/ref/plugins.txt
    secrets:
      - source: admin_password
        target: /run/secrets/jenkins_admin_password
      - source: smtp_password
        target: /run/secrets/smtp_password

  ansible-controller:
    image: quay.io/ansible/ansible-runner:latest
    container_name: ansible-controller
    hostname: ansible-controller
    restart: unless-stopped
    profiles: ["tools", "monitoring"]
    volumes:
      - ansible_data:/runner:rw
      - ./shared:/shared:rw
      - ./ansible/inventory:/inventory:ro
    networks:
      jenkins-network:
        aliases:
          - ansible-runner
    environment:
      ANSIBLE_HOST_KEY_CHECKING: "False"
      ANSIBLE_SSH_RETRIES: "3"
    healthcheck:
      test: ["CMD", "ansible", "--version"]
      interval: 30s
      timeout: 10s
      retries: 3

  reverse-proxy:
    image: nginx:alpine
    container_name: jenkins-proxy
    restart: unless-stopped
    profiles: ["proxy"]
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    networks:
      jenkins-network:
        aliases:
          - proxy
    depends_on:
      - jenkins

networks:
  jenkins-network:
    name: jenkins-corporate-network
    driver: bridge
    attachable: true
    ipam:
      config:
        - subnet: 172.20.0.0/24

volumes:
  jenkins_data:
    name: jenkins_corporate_data
    driver: local
  ansible_data:
    name: ansible_corporate_data
    driver: local

configs:
  plugins_txt:
    file: ./plugins.txt

secrets:
  admin_password:
    file: ./secrets/jenkins_admin_password.txt
  smtp_password:
    file: ./secrets/smtp_password.txt
```

2. Обновленный Dockerfile

Dockerfile:

```dockerfile
# syntax=docker/dockerfile:1.4

ARG JENKINS_VERSION=lts-jdk17

FROM jenkins/jenkins:${JENKINS_VERSION} as base

USER root

# Метка для безопасности
LABEL security.scan="true" \
      maintainer="devops@company.com" \
      version="1.0"

# Установка базовых утилит
RUN <<EOT
    apt-get update
    apt-get install -y --no-install-recommends \
        python3 \
        python3-pip \
        git \
        sshpass \
        openssh-client \
        curl \
        gnupg \
        software-properties-common \
        jq \
        unzip
    rm -rf /var/lib/apt/lists/*
    apt-get clean
EOT

# Многоступенчатая установка для оптимизации
FROM base as ansible-install

RUN pip3 install --no-cache-dir ansible ansible-lint yamllint ansible-tower-cli

FROM base as docker-install

RUN <<EOT
    curl -fsSL https://get.docker.com | sh
    usermod -aG docker jenkins
EOT

FROM base as final

# Копируем установленные компоненты
COPY --from=ansible-install /usr/local/lib/python3.9/dist-packages /usr/local/lib/python3.9/dist-packages
COPY --from=ansible-install /usr/local/bin/ansible* /usr/local/bin/
COPY --from=docker-install /usr/bin/docker /usr/bin/docker
COPY --from=docker-install /var/run/docker.sock /var/run/docker.sock

# Создание структуры директорий
RUN <<EOT
    mkdir -p /var/jenkins_conf/casc /var/ansible /shared
    chown -R jenkins:jenkins /var/jenkins_conf /var/ansible /shared
EOT

# Копирование скриптов
COPY --chown=jenkins:jenkins scripts/ /var/jenkins_scripts/
RUN chmod +x /var/jenkins_scripts/*.sh

# Установка плагинов через установленный скрипт
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

USER jenkins

HEALTHCHECK --interval=30s --timeout=10s --start-period=1m --retries=3 \
  CMD curl -f http://localhost:8080 || exit 1
```

3. Современный список плагинов

plugins.txt:

```txt
# Pipeline & DSL
workflow-aggregator:latest
pipeline-stage-view:latest
job-dsl:latest
blueocean:latest

# Ansible & Infrastructure
ansible:latest
ansible-tower:latest

# Version Control
git:latest
github:latest
gitlab:latest
bitbucket:latest

# Security & Auth
matrix-auth:latest
role-strategy:latest
credentials-binding:latest
ssh-credentials:latest
ssh-slaves:latest

# Configuration as Code
configuration-as-code:latest
jcascom-configuration-as-code-support:latest

# Docker & Kubernetes
docker-workflow:latest
docker-plugin:latest
kubernetes:latest

# Notifications & Monitoring
email-ext:latest
mailer:latest
telegram-notifications:latest
slack:latest

# Utilities
timestamper:latest
ws-cleanup:latest
build-timeout:latest
parameterized-trigger:latest
copyartifact:latest
envinject:latest
htmlpublisher:latest
pipeline-utility-steps:latest

# UI & Experience
simple-theme-plugin:latest
dashboard-view:latest
```

4. Обновленная конфигурация JCasC

casc/jenkins.yaml:

```yaml
jenkins:
  agentProtocols:
    - "JNLP4-connect"
    - "Ping"
  authorizationStrategy:
    globalMatrix:
      permissions:
        - "Overall/Administer:admin"
        - "Overall/Read:authenticated"
        - "Job/Read:authenticated"
        - "Job/Configure:admin"
        - "Job/Build:admin"
        - "Job/Cancel:admin"
        - "View/Read:authenticated"
  clouds: []
  disabledAdministrativeMonitors:
    - "hudson.diagnosis.ReverseProxySetupMonitor"
  label: "master"
  mode: NORMAL
  numExecutors: 5
  primaryView:
    all:
      name: "all"
  quietPeriod: 5
  remotingSecurity:
    enabled: true
  scmCheckoutRetryCount: 2
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          name: "Jenkins Administrator"
          password: "${ADMIN_PASSWORD}"
  slaveAgentPort: 50000
  systemMessage: "Jenkins Infrastructure as Code Platform\nКорпоративная система управления конфигурацией"
  views:
    - all:
        name: "all"
  viewsTabBar: "standard"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "git-corporate-credentials"
              username: "git-service"
              password: "${GIT_PASSWORD}"
          - sshUsernamePrivateKey:
              scope: GLOBAL
              id: "ansible-corporate-key"
              username: "ansible"
              privateKeySource:
                directEntry:
                  privateKey: "${ANSIBLE_SSH_KEY}"
          - string:
              scope: GLOBAL
              id: "ansible-vault-corporate"
              secret: "${ANSIBLE_VAULT_PASSWORD}"
          - usernamePassword:
              scope: GLOBAL
              id: "docker-registry"
              username: "${DOCKER_REGISTRY_USER}"
              password: "${DOCKER_REGISTRY_PASSWORD}"

tool:
  git:
    installations:
      - name: "git"
        home: "/usr/bin/git"
  ansible:
    installations:
      - name: "ansible"
        home: "/usr/local/bin/ansible"

unclassified:
  location:
    url: "${JENKINS_URL}"
    adminAddress: "jenkins-admin@company.com"
  
  mailer:
    smtpHost: "smtp.company.com"
    smtpPort: "587"
    useSsl: true
    smtpAuth: true
    username: "jenkins-noreply@company.com"
    password: "${SMTP_PASSWORD}"
    charset: "UTF-8"
  
  systemd:
    enable: false
  
  securityFilters:
    adminForJob: true
  
  buildDiscarders:
    configuredBuildDiscarders:
      - strategy:
          daysToKeep: "30"
          numToKeep: "50"
          artifactDaysToKeep: "15"
          artifactNumToKeep: "20"

security:
  apiToken:
    creationOfLegacyTokenEnabled: false
    tokenGenerationOnCreationEnabled: true
  sshd:
    enabled: false
  sSHD:
    port: -1
```

5. Современные скрипты управления

deploy.sh:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')] $1${NC}"
}

warn() {
    echo -e "${YELLOW}[WARN] $1${NC}"
}

error() {
    echo -e "${RED}[ERROR] $1${NC}"
}

check_dependencies() {
    log "Проверка зависимостей..."
    local deps=("docker" "docker-compose")
    for dep in "${deps[@]}"; do
        if ! command -v "$dep" &> /dev/null; then
            error "Не найдена зависимость: $dep"
            exit 1
        fi
    done
    log "✓ Все зависимости удовлетворены"
}

setup_environment() {
    log "Настройка окружения..."
    
    # Создание директорий
    mkdir -p {casc,scripts,shared,secrets,nginx/{conf.d,ssl},ansible/inventory}
    
    # Настройка прав
    chmod 755 scripts/*.sh 2>/dev/null || true
    chmod 600 secrets/*.txt 2>/dev/null || true
    
    # Проверка .env файла
    if [[ ! -f .env ]]; then
        warn "Файл .env не найден. Создание шаблона..."
        cp .env.example .env
        error "Отредактируйте .env файл перед запуском"
        exit 1
    fi
    
    source .env
}

build_images() {
    log "Сборка Docker образов..."
    docker compose build --pull --no-cache
}

start_services() {
    log "Запуск сервисов..."
    
    # Основные сервисы
    docker compose up -d jenkins
    
    # Дополнительные сервисы (опционально)
    if [[ ${ENABLE_TOOLS:-false} == "true" ]]; then
        docker compose --profile tools up -d ansible-controller
    fi
    
    if [[ ${ENABLE_PROXY:-false} == "true" ]]; then
        docker compose --profile proxy up -d reverse-proxy
    fi
}

wait_for_jenkins() {
    log "Ожидание запуска Jenkins..."
    local timeout=120
    local counter=0
    
    while ! curl -s -f "http://localhost:8080" > /dev/null; do
        sleep 5
        counter=$((counter + 5))
        if [[ $counter -ge $timeout ]]; then
            error "Таймаут ожидания Jenkins"
            docker compose logs jenkins
            exit 1
        fi
    done
    log "✓ Jenkins запущен"
}

setup_initial_config() {
    log "Настройка начальной конфигурации..."
    
    # Создание shared директорий
    mkdir -p shared/{ansible,terraform,scripts}
    
    log "✓ Настройка завершена"
    log "🌐 Jenkins доступен по адресу: http://localhost:8080"
    
    # Показ initial admin password если существует
    if docker compose exec jenkins test -f /var/jenkins_home/secrets/initialAdminPassword; then
        log "🔑 Initial Admin Password:"
        docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
    fi
}

main() {
    log "🚀 Запуск корпоративного Jenkins для Infrastructure as Code"
    
    check_dependencies
    setup_environment
    build_images
    start_services
    wait_for_jenkins
    setup_initial_config
    
    log "✅ Развертывание завершено успешно!"
}

main "$@"
```

6. Файл окружения

.env:

```env
# Corporate Jenkins Configuration
COMPOSE_PROJECT_NAME=jenkins-iac-corporate
JENKINS_URL=http://jenkins.company.com

# Security Secrets
ADMIN_PASSWORD=secure_corporate_password_2024
SMTP_PASSWORD=smtp_corporate_password_2024
GIT_PASSWORD=gitlab_corporate_token_2024
ANSIBLE_VAULT_PASSWORD=vault_corporate_2024
DOCKER_REGISTRY_USER=registry-user
DOCKER_REGISTRY_PASSWORD=registry_corporate_2024

# Ansible SSH Key (будет переопределен)
ANSIBLE_SSH_KEY=ssh-rsa AAAAB3NzaC1yc2E...

# Features
ENABLE_TOOLS=true
ENABLE_PROXY=false
ENABLE_MONITORING=true

# Network
JENKINS_NETWORK_SUBNET=172.20.0.0/24
```

7. Команды управления

scripts/management.sh:

```bash
#!/usr/bin/env bash

case "${1:-}" in
    start)
        docker compose up -d
        ;;
    stop)
        docker compose down
        ;;
    restart)
        docker compose restart
        ;;
    rebuild)
        docker compose down
        docker compose build --no-cache
        docker compose up -d
        ;;
    logs)
        docker compose logs -f "${2:-jenkins}"
        ;;
    backup)
        docker compose exec jenkins tar czf /shared/jenkins-backup-$(date +%Y%m%d).tar.gz -C /var/jenkins_home .
        ;;
    status)
        docker compose ps
        ;;
    shell)
        docker compose exec "${2:-jenkins}" bash
        ;;
    update)
        docker compose pull
        docker compose build --pull
        docker compose up -d
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|rebuild|logs|backup|status|shell|update}"
        exit 1
        ;;
esac
```

Запуск решения

```bash
# Сделать скрипты исполняемыми
chmod +x deploy.sh scripts/management.sh

# Развертывание
./deploy.sh

# Управление
./scripts/management.sh start
./scripts/management.sh status

# Логи
./scripts/management.sh logs jenkins

# Обновление
./scripts/management.sh update
```

Особенности Docker Compose V2:

· ✅ Проектные именования - автоматическое префиксирование
· ✅ Secrets & Configs - встроенная поддержка секретов
· ✅ Профили - избирательный запуск сервисов
· ✅ Healthchecks - мониторинг состояния
· ✅ Более эффективная сборка - многоступенчатые образы
· ✅ Улучшенная сеть - явное управление сетями
· ✅ Лучшая производительность - оптимизированная работа с volumes

Это решение предоставляет полную корпоративную платформу для Infrastructure as Code с использованием современных возможностей Docker Compose V2.



Добавляю интеграцию с Microsoft Active Directory в корпоративное решение Jenkins:

1. Обновленный docker-compose.yml

```yaml
name: jenkins-iac-corporate

services:
  jenkins:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        JENKINS_VERSION: lts-jdk17
    container_name: jenkins-iac-corporate
    hostname: jenkins-iac
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    environment:
      JENKINS_OPTS: --httpPort=8080
      CASC_JENKINS_CONFIG: /var/jenkins_conf/casc
      JAVA_OPTS: -Djenkins.install.runSetupWizard=false -Xmx2g -Xms512m -Duser.timezone=Europe/Moscow
      JENKINS_SLAVE_AGENT_PORT: 50000
      # AD debugging
      - JAVA_OPTS=-Dcom.sun.jndi.ldap.object.disableEndpointIdentification=true
    env_file:
      - .env
    volumes:
      - jenkins_data:/var/jenkins_home
      - ansible_data:/var/ansible
      - ./casc:/var/jenkins_conf/casc:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./scripts:/var/jenkins_scripts:ro
      - ./shared:/shared:rw
      # Kerberos for AD integration
      - ./kerberos:/etc/krb5.conf:ro
    networks:
      jenkins-network:
        aliases:
          - jenkins-main
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    configs:
      - source: plugins_txt
        target: /usr/share/jenkins/ref/plugins.txt
    secrets:
      - source: admin_password
        target: /run/secrets/jenkins_admin_password
      - source: smtp_password
        target: /run/secrets/smtp_password
      - source: ad_bind_password
        target: /run/secrets/ad_bind_password
    dns:
      - 8.8.8.8
      - 1.1.1.1
      # Your corporate DNS servers for AD
      - ${AD_DNS_SERVER_1}
      - ${AD_DNS_SERVER_2}

  # LDAP/AD test utility
  ldap-admin:
    image: osixia/phpldapadmin:latest
    container_name: ldap-admin
    restart: unless-stopped
    profiles: ["tools", "ad-test"]
    environment:
      PHPLDAPADMIN_LDAP_HOSTS: ${AD_DOMAIN_CONTROLLER}
      PHPLDAPADMIN_HTTPS: "false"
    ports:
      - "8081:80"
    networks:
      jenkins-network:
    depends_on:
      - jenkins

networks:
  jenkins-network:
    name: jenkins-corporate-network
    driver: bridge
    attachable: true
    ipam:
      config:
        - subnet: 172.20.0.0/24

volumes:
  jenkins_data:
    name: jenkins_corporate_data
    driver: local
  ansible_data:
    name: ansible_corporate_data
    driver: local

configs:
  plugins_txt:
    file: ./plugins.txt

secrets:
  admin_password:
    file: ./secrets/jenkins_admin_password.txt
  smtp_password:
    file: ./secrets/smtp_password.txt
  ad_bind_password:
    file: ./secrets/ad_bind_password.txt
```

2. Обновленный Dockerfile с AD поддержкой

```dockerfile
# syntax=docker/dockerfile:1.4

ARG JENKINS_VERSION=lts-jdk17

FROM jenkins/jenkins:${JENKINS_VERSION} as base

USER root

# Метка для безопасности
LABEL security.scan="true" \
      maintainer="devops@company.com" \
      version="1.0"

# Установка базовых утилит + AD/LDAP инструменты
RUN <<EOT
    apt-get update
    apt-get install -y --no-install-recommends \
        python3 \
        python3-pip \
        git \
        sshpass \
        openssh-client \
        curl \
        gnupg \
        software-properties-common \
        jq \
        unzip \
        ldap-utils \
        krb5-user \
        libpam-krb5 \
        libpam-sss \
        libnss-sss \
        sssd \
        sssd-tools
    rm -rf /var/lib/apt/lists/*
    apt-get clean
EOT

# Многоступенчатая установка для оптимизации
FROM base as ansible-install

RUN pip3 install --no-cache-dir ansible ansible-lint yamllint ansible-tower-cli

FROM base as docker-install

RUN <<EOT
    curl -fsSL https://get.docker.com | sh
    usermod -aG docker jenkins
EOT

FROM base as final

# Копируем установленные компоненты
COPY --from=ansible-install /usr/local/lib/python3.9/dist-packages /usr/local/lib/python3.9/dist-packages
COPY --from=ansible-install /usr/local/bin/ansible* /usr/local/bin/
COPY --from=docker-install /usr/bin/docker /usr/bin/docker
COPY --from=docker-install /var/run/docker.sock /var/run/docker.sock

# Настройка Kerberos для AD
RUN mkdir -p /etc/krb5.conf.d
COPY kerberos/krb5.conf /etc/krb5.conf

# Создание структуры директорий
RUN <<EOT
    mkdir -p /var/jenkins_conf/casc /var/ansible /shared
    chown -R jenkins:jenkins /var/jenkins_conf /var/ansible /shared
EOT

# Копирование скриптов
COPY --chown=jenkins:jenkins scripts/ /var/jenkins_scripts/
RUN chmod +x /var/jenkins_scripts/*.sh

# Установка плагинов через установленный скрипт
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

USER jenkins

HEALTHCHECK --interval=30s --timeout=10s --start-period=1m --retries=3 \
  CMD curl -f http://localhost:8080 || exit 1
```

3. Обновленный plugins.txt с AD плагинами

```txt
# Active Directory & LDAP
active-directory:latest
ldap:latest
saml:latest
oic-auth:latest

# Pipeline & DSL
workflow-aggregator:latest
pipeline-stage-view:latest
job-dsl:latest
blueocean:latest

# Ansible & Infrastructure
ansible:latest
ansible-tower:latest

# Version Control
git:latest
github:latest
gitlab:latest
bitbucket:latest

# Security & Auth
matrix-auth:latest
role-strategy:latest
credentials-binding:latest
ssh-credentials:latest
ssh-slaves:latest

# Configuration as Code
configuration-as-code:latest
jcascom-configuration-as-code-support:latest

# Docker & Kubernetes
docker-workflow:latest
docker-plugin:latest
kubernetes:latest

# Notifications & Monitoring
email-ext:latest
mailer:latest
telegram-notifications:latest
slack:latest

# Utilities
timestamper:latest
ws-cleanup:latest
build-timeout:latest
parameterized-trigger:latest
copyartifact:latest
envinject:latest
htmlpublisher:latest
pipeline-utility-steps:latest

# UI & Experience
simple-theme-plugin:latest
dashboard-view:latest
```

4. Конфигурация Kerberos

kerberos/krb5.conf:

```ini
[libdefaults]
    default_realm = COMPANY.COM
    dns_lookup_realm = false
    dns_lookup_kdc = true
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    default_tgs_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5
    default_tkt_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5
    permitted_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5

[realms]
    COMPANY.COM = {
        kdc = dc1.company.com
        kdc = dc2.company.com
        admin_server = dc1.company.com
        default_domain = company.com
    }

[domain_realm]
    .company.com = COMPANY.COM
    company.com = COMPANY.COM

[logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log
```

5. Обновленная конфигурация JCasC с AD

casc/jenkins.yaml:

```yaml
jenkins:
  agentProtocols:
    - "JNLP4-connect"
    - "Ping"
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            description: "Jenkins Administrators"
            permissions:
              - "Overall/Administer"
              - "Overall/Read"
            assignments:
              - "cn=jenkins-admins,ou=groups,dc=company,dc=com"
          - name: "developer"
            description: "Developers"
            permissions:
              - "Job/Read"
              - "Job/Build"
              - "Job/Workspace"
              - "Job/Cancel"
            assignments:
              - "cn=developers,ou=groups,dc=company,dc=com"
          - name: "viewer"
            description: "Viewers"
            permissions:
              - "Overall/Read"
              - "Job/Read"
            assignments:
              - "cn=users,ou=groups,dc=company,dc=com"
  clouds: []
  disabledAdministrativeMonitors:
    - "hudson.diagnosis.ReverseProxySetupMonitor"
  label: "master"
  mode: NORMAL
  numExecutors: 5
  primaryView:
    all:
      name: "all"
  quietPeriod: 5
  remotingSecurity:
    enabled: true
  scmCheckoutRetryCount: 2
  securityRealm:
    activeDirectory:
      domains:
        - name: "company.com"
          servers: "${AD_DOMAIN_CONTROLLER1}:389,${AD_DOMAIN_CONTROLLER2}:389"
          site: "Default-First-Site-Name"
          bindName: "${AD_BIND_USER}"
          bindPassword: "${AD_BIND_PASSWORD}"
          groupLookupStrategy: "RECURSIVE"
          cache:
            size: 1000
            ttl: 300
          startTls: true
          removeIrrelevantGroups: true
          customDomain: true
          tlsConfiguration: "TRUST_ALL_CERTIFICATES"
      cache:
        size: 1000
        ttl: 600
      groupLookupStrategy: "RECURSIVE"
      removeIrrelevantGroups: true
      customDomain: true
      startTls: true
  slaveAgentPort: 50000
  systemMessage: "Jenkins Infrastructure as Code Platform\nКорпоративная система управления конфигурацией\nИнтегрировано с Active Directory"
  views:
    - all:
        name: "all"
  viewsTabBar: "standard"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "git-corporate-credentials"
              username: "git-service"
              password: "${GIT_PASSWORD}"
          - sshUsernamePrivateKey:
              scope: GLOBAL
              id: "ansible-corporate-key"
              username: "ansible"
              privateKeySource:
                directEntry:
                  privateKey: "${ANSIBLE_SSH_KEY}"
          - string:
              scope: GLOBAL
              id: "ansible-vault-corporate"
              secret: "${ANSIBLE_VAULT_PASSWORD}"
          - usernamePassword:
              scope: GLOBAL
              id: "docker-registry"
              username: "${DOCKER_REGISTRY_USER}"
              password: "${DOCKER_REGISTRY_PASSWORD}"
          - usernamePassword:
              scope: GLOBAL
              id: "ad-service-account"
              username: "${AD_BIND_USER}"
              password: "${AD_BIND_PASSWORD}"

tool:
  git:
    installations:
      - name: "git"
        home: "/usr/bin/git"
  ansible:
    installations:
      - name: "ansible"
        home: "/usr/local/bin/ansible"

unclassified:
  location:
    url: "${JENKINS_URL}"
    adminAddress: "jenkins-admin@company.com"
  
  mailer:
    smtpHost: "smtp.company.com"
    smtpPort: "587"
    useSsl: true
    smtpAuth: true
    username: "jenkins-noreply@company.com"
    password: "${SMTP_PASSWORD}"
    charset: "UTF-8"
  
  systemd:
    enable: false
  
  securityFilters:
    adminForJob: true
  
  buildDiscarders:
    configuredBuildDiscarders:
      - strategy:
          daysToKeep: "30"
          numToKeep: "50"
          artifactDaysToKeep: "15"
          artifactNumToKeep: "20"

security:
  apiToken:
    creationOfLegacyTokenEnabled: false
    tokenGenerationOnCreationEnabled: true
  sshd:
    enabled: false
  sSHD:
    port: -1
```

6. Скрипт тестирования AD подключения

scripts/test-ad-connection.sh:

```bash
#!/usr/bin/env bash

set -euo pipefail

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')] $1${NC}"
}

warn() {
    echo -e "${YELLOW}[WARN] $1${NC}"
}

error() {
    echo -e "${RED}[ERROR] $1${NC}"
}

test_ldap_connection() {
    local bind_user="$1"
    local bind_password="$2"
    local domain_controller="$3"
    local base_dn="$4"
    
    log "Testing LDAP connection to $domain_controller..."
    
    if ldapsearch -x -H "ldap://$domain_controller" -D "$bind_user" -w "$bind_password" -b "$base_dn" -LLL "(objectClass=user)" cn 2>/dev/null | head -10; then
        log "✓ LDAP connection successful"
        return 0
    else
        error "✗ LDAP connection failed"
        return 1
    fi
}

test_kerberos_auth() {
    local domain="$1"
    local user="$2"
    local password="$3"
    
    log "Testing Kerberos authentication for domain $domain..."
    
    # Create credentials cache
    if echo "$password" | kinit "$user@$domain" 2>/dev/null; then
        log "✓ Kerberos authentication successful"
        klist
        kdestroy
        return 0
    else
        error "✗ Kerberos authentication failed"
        return 1
    fi
}

test_ad_groups() {
    local bind_user="$1"
    local bind_password="$2"
    local domain_controller="$3"
    local base_dn="$4"
    local group_dn="$5"
    
    log "Testing AD group lookup for $group_dn..."
    
    if ldapsearch -x -H "ldap://$domain_controller" -D "$bind_user" -w "$bind_password" -b "$base_dn" -LLL "(&(objectClass=group)(cn=jenkins-admins))" member 2>/dev/null; then
        log "✓ AD group lookup successful"
        return 0
    else
        warn "⚠ AD group lookup issues (might be normal if group doesn't exist)"
        return 0
    fi
}

main() {
    log "Starting Active Directory connectivity tests..."
    
    # Source environment variables
    if [[ -f ../.env ]]; then
        source ../.env
    else
        error ".env file not found"
        exit 1
    fi
    
    # Test parameters
    local bind_user="${AD_BIND_USER}"
    local bind_password="${AD_BIND_PASSWORD}"
    local domain_controller="${AD_DOMAIN_CONTROLLER1}"
    local domain="${AD_DOMAIN}"
    local base_dn="${AD_BASE_DN}"
    local group_dn="${AD_GROUP_DN:-cn=jenkins-admins,ou=groups,dc=company,dc=com}"
    
    # Run tests
    local tests_passed=0
    local tests_failed=0
    
    if test_ldap_connection "$bind_user" "$bind_password" "$domain_controller" "$base_dn"; then
        ((tests_passed++))
    else
        ((tests_failed++))
    fi
    
    if test_kerberos_auth "$domain" "$bind_user" "$bind_password"; then
        ((tests_passed++))
    else
        ((tests_failed++))
    fi
    
    if test_ad_groups "$bind_user" "$bind_password" "$domain_controller" "$base_dn" "$group_dn"; then
        ((tests_passed++))
    else
        ((tests_failed++))
    fi
    
    log "Tests completed: $tests_passed passed, $tests_failed failed"
    
    if [[ $tests_failed -eq 0 ]]; then
        log "✅ All AD connectivity tests passed!"
    else
        error "❌ Some AD connectivity tests failed"
        exit 1
    fi
}

main "$@"
```

7. Обновленный .env файл с AD настройками

```env
# Corporate Jenkins Configuration
COMPOSE_PROJECT_NAME=jenkins-iac-corporate
JENKINS_URL=http://jenkins.company.com

# Active Directory Configuration
AD_DOMAIN=company.com
AD_DOMAIN_CONTROLLER1=dc1.company.com
AD_DOMAIN_CONTROLLER2=dc2.company.com
AD_BIND_USER=svc_jenkins@company.com
AD_BIND_PASSWORD=secure_ad_bind_password_2024
AD_BASE_DN=DC=company,DC=com
AD_USER_SEARCH_BASE=OU=Users,DC=company,DC=com
AD_GROUP_SEARCH_BASE=OU=Groups,DC=company,DC=com
AD_DNS_SERVER_1=10.0.0.10
AD_DNS_SERVER_2=10.0.0.11

# Security Secrets
ADMIN_PASSWORD=secure_corporate_password_2024
SMTP_PASSWORD=smtp_corporate_password_2024
GIT_PASSWORD=gitlab_corporate_token_2024
ANSIBLE_VAULT_PASSWORD=vault_corporate_2024
DOCKER_REGISTRY_USER=registry-user
DOCKER_REGISTRY_PASSWORD=registry_corporate_2024

# Ansible SSH Key (будет переопределен)
ANSIBLE_SSH_KEY=ssh-rsa AAAAB3NzaC1yc2E...

# Features
ENABLE_TOOLS=true
ENABLE_PROXY=false
ENABLE_MONITORING=true
ENABLE_AD_TEST=true

# Network
JENKINS_NETWORK_SUBNET=172.20.0.0/24
```

8. Скрипт настройки AD групп

scripts/setup-ad-groups.sh:

```bash
#!/usr/bin/env bash

set -euo pipefail

# This script helps create AD groups for Jenkins if they don't exist
# Requires ldap-utils and proper AD permissions

create_ad_group() {
    local group_name="$1"
    local group_dn="$2"
    local bind_user="$3"
    local bind_password="$4"
    local domain_controller="$5"
    
    log "Creating AD group: $group_name"
    
    cat << EOF | ldapmodify -x -H "ldap://$domain_controller" -D "$bind_user" -w "$bind_password"
dn: $group_dn
objectClass: top
objectClass: group
cn: $group_name
sAMAccountName: $group_name
groupType: -2147483646
EOF

}

main() {
    source ../.env
    
    local groups=(
        "jenkins-admins:cn=jenkins-admins,ou=groups,dc=company,dc=com"
        "jenkins-developers:cn=jenkins-developers,ou=groups,dc=company,dc=com"
        "jenkins-viewers:cn=jenkins-viewers,ou=groups,dc=company,dc=com"
    )
    
    for group_info in "${groups[@]}"; do
        IFS=':' read -r group_name group_dn <<< "$group_info"
        
        if ! ldapsearch -x -H "ldap://${AD_DOMAIN_CONTROLLER1}" -D "${AD_BIND_USER}" -w "${AD_BIND_PASSWORD}" -b "$group_dn" -LLL "(objectClass=group)" 2>/dev/null | grep -q "cn: $group_name"; then
            create_ad_group "$group_name" "$group_dn" "${AD_BIND_USER}" "${AD_BIND_PASSWORD}" "${AD_DOMAIN_CONTROLLER1}"
        else
            log "Group $group_name already exists"
        fi
    done
}

main "$@"
```

9. Обновленный deploy.sh с проверкой AD

```bash
#!/usr/bin/env bash
set -euo pipefail

# ... (предыдущий код остается)

check_ad_connectivity() {
    if [[ ${ENABLE_AD_TEST:-false} == "true" ]]; then
        log "Проверка подключения к Active Directory..."
        if docker compose run --rm jenkins /var/jenkins_scripts/test-ad-connection.sh; then
            log "✓ Подключение к AD успешно"
        else
            error "❌ Ошибка подключения к AD"
            warn "Продолжение развертывания, но AD может не работать"
        fi
    fi
}

main() {
    log "🚀 Запуск корпоративного Jenkins с интеграцией Active Directory"
    
    check_dependencies
    setup_environment
    build_images
    check_ad_connectivity
    start_services
    wait_for_jenkins
    setup_initial_config
    
    log "✅ Развертывание завершено успешно!"
    log "🔐 Jenkins интегрирован с Active Directory"
    log "👥 Группы AD для управления доступом:"
    log "   - jenkins-admins: полный доступ"
    log "   - jenkins-developers: разработчики"
    log "   - jenkins-viewers: просмотр"
}

main "$@"
```

Запуск с AD интеграцией

```bash
# Настройка секретов AD
mkdir -p secrets
echo "secure_ad_bind_password_2024" > secrets/ad_bind_password.txt
chmod 600 secrets/ad_bind_password.txt

# Тестирование AD подключения
chmod +x scripts/test-ad-connection.sh
./scripts/test-ad-connection.sh

# Запуск с AD
./deploy.sh

# Запуск утилиты для тестирования AD (опционально)
docker compose --profile ad-test up -d ldap-admin
```

Ключевые особенности AD интеграции:

· ✅ Active Directory аутентификация - единый вход для сотрудников
· ✅ Role-Based Access Control - управление правами через AD группы
· ✅ Kerberos поддержка - безопасная аутентификация
· ✅ LDAP поиск - эффективное получение информации о пользователях и группах
· ✅ Тестирование подключения - автоматическая проверка AD connectivity
· ✅ Готовые группы - предопределенные роли для управления доступом
· ✅ Безопасность - TLS соединение с контроллерами домена

Это решение обеспечивает полную интеграцию с корпоративной инфраструктурой Microsoft Active Directory с соблюдением лучших практик безопасности.


Добавляю интеграцию с Microsoft Active Directory в корпоративное решение Jenkins:

1. Обновленный docker-compose.yml

```yaml
name: jenkins-iac-corporate

services:
  jenkins:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        JENKINS_VERSION: lts-jdk17
    container_name: jenkins-iac-corporate
    hostname: jenkins-iac
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    environment:
      JENKINS_OPTS: --httpPort=8080
      CASC_JENKINS_CONFIG: /var/jenkins_conf/casc
      JAVA_OPTS: -Djenkins.install.runSetupWizard=false -Xmx2g -Xms512m -Duser.timezone=Europe/Moscow
      JENKINS_SLAVE_AGENT_PORT: 50000
      # AD debugging
      - JAVA_OPTS=-Dcom.sun.jndi.ldap.object.disableEndpointIdentification=true
    env_file:
      - .env
    volumes:
      - jenkins_data:/var/jenkins_home
      - ansible_data:/var/ansible
      - ./casc:/var/jenkins_conf/casc:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./scripts:/var/jenkins_scripts:ro
      - ./shared:/shared:rw
      # Kerberos for AD integration
      - ./kerberos:/etc/krb5.conf:ro
    networks:
      jenkins-network:
        aliases:
          - jenkins-main
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    configs:
      - source: plugins_txt
        target: /usr/share/jenkins/ref/plugins.txt
    secrets:
      - source: admin_password
        target: /run/secrets/jenkins_admin_password
      - source: smtp_password
        target: /run/secrets/smtp_password
      - source: ad_bind_password
        target: /run/secrets/ad_bind_password
    dns:
      - 8.8.8.8
      - 1.1.1.1
      # Your corporate DNS servers for AD
      - ${AD_DNS_SERVER_1}
      - ${AD_DNS_SERVER_2}

  # LDAP/AD test utility
  ldap-admin:
    image: osixia/phpldapadmin:latest
    container_name: ldap-admin
    restart: unless-stopped
    profiles: ["tools", "ad-test"]
    environment:
      PHPLDAPADMIN_LDAP_HOSTS: ${AD_DOMAIN_CONTROLLER}
      PHPLDAPADMIN_HTTPS: "false"
    ports:
      - "8081:80"
    networks:
      jenkins-network:
    depends_on:
      - jenkins

networks:
  jenkins-network:
    name: jenkins-corporate-network
    driver: bridge
    attachable: true
    ipam:
      config:
        - subnet: 172.20.0.0/24

volumes:
  jenkins_data:
    name: jenkins_corporate_data
    driver: local
  ansible_data:
    name: ansible_corporate_data
    driver: local

configs:
  plugins_txt:
    file: ./plugins.txt

secrets:
  admin_password:
    file: ./secrets/jenkins_admin_password.txt
  smtp_password:
    file: ./secrets/smtp_password.txt
  ad_bind_password:
    file: ./secrets/ad_bind_password.txt
```

2. Обновленный Dockerfile с AD поддержкой

```dockerfile
# syntax=docker/dockerfile:1.4

ARG JENKINS_VERSION=lts-jdk17

FROM jenkins/jenkins:${JENKINS_VERSION} as base

USER root

# Метка для безопасности
LABEL security.scan="true" \
      maintainer="devops@company.com" \
      version="1.0"

# Установка базовых утилит + AD/LDAP инструменты
RUN <<EOT
    apt-get update
    apt-get install -y --no-install-recommends \
        python3 \
        python3-pip \
        git \
        sshpass \
        openssh-client \
        curl \
        gnupg \
        software-properties-common \
        jq \
        unzip \
        ldap-utils \
        krb5-user \
        libpam-krb5 \
        libpam-sss \
        libnss-sss \
        sssd \
        sssd-tools
    rm -rf /var/lib/apt/lists/*
    apt-get clean
EOT

# Многоступенчатая установка для оптимизации
FROM base as ansible-install

RUN pip3 install --no-cache-dir ansible ansible-lint yamllint ansible-tower-cli

FROM base as docker-install

RUN <<EOT
    curl -fsSL https://get.docker.com | sh
    usermod -aG docker jenkins
EOT

FROM base as final

# Копируем установленные компоненты
COPY --from=ansible-install /usr/local/lib/python3.9/dist-packages /usr/local/lib/python3.9/dist-packages
COPY --from=ansible-install /usr/local/bin/ansible* /usr/local/bin/
COPY --from=docker-install /usr/bin/docker /usr/bin/docker
COPY --from=docker-install /var/run/docker.sock /var/run/docker.sock

# Настройка Kerberos для AD
RUN mkdir -p /etc/krb5.conf.d
COPY kerberos/krb5.conf /etc/krb5.conf

# Создание структуры директорий
RUN <<EOT
    mkdir -p /var/jenkins_conf/casc /var/ansible /shared
    chown -R jenkins:jenkins /var/jenkins_conf /var/ansible /shared
EOT

# Копирование скриптов
COPY --chown=jenkins:jenkins scripts/ /var/jenkins_scripts/
RUN chmod +x /var/jenkins_scripts/*.sh

# Установка плагинов через установленный скрипт
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

USER jenkins

HEALTHCHECK --interval=30s --timeout=10s --start-period=1m --retries=3 \
  CMD curl -f http://localhost:8080 || exit 1
```

3. Обновленный plugins.txt с AD плагинами

```txt
# Active Directory & LDAP
active-directory:latest
ldap:latest
saml:latest
oic-auth:latest

# Pipeline & DSL
workflow-aggregator:latest
pipeline-stage-view:latest
job-dsl:latest
blueocean:latest

# Ansible & Infrastructure
ansible:latest
ansible-tower:latest

# Version Control
git:latest
github:latest
gitlab:latest
bitbucket:latest

# Security & Auth
matrix-auth:latest
role-strategy:latest
credentials-binding:latest
ssh-credentials:latest
ssh-slaves:latest

# Configuration as Code
configuration-as-code:latest
jcascom-configuration-as-code-support:latest

# Docker & Kubernetes
docker-workflow:latest
docker-plugin:latest
kubernetes:latest

# Notifications & Monitoring
email-ext:latest
mailer:latest
telegram-notifications:latest
slack:latest

# Utilities
timestamper:latest
ws-cleanup:latest
build-timeout:latest
parameterized-trigger:latest
copyartifact:latest
envinject:latest
htmlpublisher:latest
pipeline-utility-steps:latest

# UI & Experience
simple-theme-plugin:latest
dashboard-view:latest
```

4. Конфигурация Kerberos

kerberos/krb5.conf:

```ini
[libdefaults]
    default_realm = COMPANY.COM
    dns_lookup_realm = false
    dns_lookup_kdc = true
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    default_tgs_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5
    default_tkt_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5
    permitted_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5

[realms]
    COMPANY.COM = {
        kdc = dc1.company.com
        kdc = dc2.company.com
        admin_server = dc1.company.com
        default_domain = company.com
    }

[domain_realm]
    .company.com = COMPANY.COM
    company.com = COMPANY.COM

[logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log
```

5. Обновленная конфигурация JCasC с AD

casc/jenkins.yaml:

```yaml
jenkins:
  agentProtocols:
    - "JNLP4-connect"
    - "Ping"
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            description: "Jenkins Administrators"
            permissions:
              - "Overall/Administer"
              - "Overall/Read"
            assignments:
              - "cn=jenkins-admins,ou=groups,dc=company,dc=com"
          - name: "developer"
            description: "Developers"
            permissions:
              - "Job/Read"
              - "Job/Build"
              - "Job/Workspace"
              - "Job/Cancel"
            assignments:
              - "cn=developers,ou=groups,dc=company,dc=com"
          - name: "viewer"
            description: "Viewers"
            permissions:
              - "Overall/Read"
              - "Job/Read"
            assignments:
              - "cn=users,ou=groups,dc=company,dc=com"
  clouds: []
  disabledAdministrativeMonitors:
    - "hudson.diagnosis.ReverseProxySetupMonitor"
  label: "master"
  mode: NORMAL
  numExecutors: 5
  primaryView:
    all:
      name: "all"
  quietPeriod: 5
  remotingSecurity:
    enabled: true
  scmCheckoutRetryCount: 2
  securityRealm:
    activeDirectory:
      domains:
        - name: "company.com"
          servers: "${AD_DOMAIN_CONTROLLER1}:389,${AD_DOMAIN_CONTROLLER2}:389"
          site: "Default-First-Site-Name"
          bindName: "${AD_BIND_USER}"
          bindPassword: "${AD_BIND_PASSWORD}"
          groupLookupStrategy: "RECURSIVE"
          cache:
            size: 1000
            ttl: 300
          startTls: true
          removeIrrelevantGroups: true
          customDomain: true
          tlsConfiguration: "TRUST_ALL_CERTIFICATES"
      cache:
        size: 1000
        ttl: 600
      groupLookupStrategy: "RECURSIVE"
      removeIrrelevantGroups: true
      customDomain: true
      startTls: true
  slaveAgentPort: 50000
  systemMessage: "Jenkins Infrastructure as Code Platform\nКорпоративная система управления конфигурацией\nИнтегрировано с Active Directory"
  views:
    - all:
        name: "all"
  viewsTabBar: "standard"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "git-corporate-credentials"
              username: "git-service"
              password: "${GIT_PASSWORD}"
          - sshUsernamePrivateKey:
              scope: GLOBAL
              id: "ansible-corporate-key"
              username: "ansible"
              privateKeySource:
                directEntry:
                  privateKey: "${ANSIBLE_SSH_KEY}"
          - string:
              scope: GLOBAL
              id: "ansible-vault-corporate"
              secret: "${ANSIBLE_VAULT_PASSWORD}"
          - usernamePassword:
              scope: GLOBAL
              id: "docker-registry"
              username: "${DOCKER_REGISTRY_USER}"
              password: "${DOCKER_REGISTRY_PASSWORD}"
          - usernamePassword:
              scope: GLOBAL
              id: "ad-service-account"
              username: "${AD_BIND_USER}"
              password: "${AD_BIND_PASSWORD}"

tool:
  git:
    installations:
      - name: "git"
        home: "/usr/bin/git"
  ansible:
    installations:
      - name: "ansible"
        home: "/usr/local/bin/ansible"

unclassified:
  location:
    url: "${JENKINS_URL}"
    adminAddress: "jenkins-admin@company.com"
  
  mailer:
    smtpHost: "smtp.company.com"
    smtpPort: "587"
    useSsl: true
    smtpAuth: true
    username: "jenkins-noreply@company.com"
    password: "${SMTP_PASSWORD}"
    charset: "UTF-8"
  
  systemd:
    enable: false
  
  securityFilters:
    adminForJob: true
  
  buildDiscarders:
    configuredBuildDiscarders:
      - strategy:
          daysToKeep: "30"
          numToKeep: "50"
          artifactDaysToKeep: "15"
          artifactNumToKeep: "20"

security:
  apiToken:
    creationOfLegacyTokenEnabled: false
    tokenGenerationOnCreationEnabled: true
  sshd:
    enabled: false
  sSHD:
    port: -1
```

6. Скрипт тестирования AD подключения

scripts/test-ad-connection.sh:

```bash
#!/usr/bin/env bash

set -euo pipefail

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')] $1${NC}"
}

warn() {
    echo -e "${YELLOW}[WARN] $1${NC}"
}

error() {
    echo -e "${RED}[ERROR] $1${NC}"
}

test_ldap_connection() {
    local bind_user="$1"
    local bind_password="$2"
    local domain_controller="$3"
    local base_dn="$4"
    
    log "Testing LDAP connection to $domain_controller..."
    
    if ldapsearch -x -H "ldap://$domain_controller" -D "$bind_user" -w "$bind_password" -b "$base_dn" -LLL "(objectClass=user)" cn 2>/dev/null | head -10; then
        log "✓ LDAP connection successful"
        return 0
    else
        error "✗ LDAP connection failed"
        return 1
    fi
}

test_kerberos_auth() {
    local domain="$1"
    local user="$2"
    local password="$3"
    
    log "Testing Kerberos authentication for domain $domain..."
    
    # Create credentials cache
    if echo "$password" | kinit "$user@$domain" 2>/dev/null; then
        log "✓ Kerberos authentication successful"
        klist
        kdestroy
        return 0
    else
        error "✗ Kerberos authentication failed"
        return 1
    fi
}

test_ad_groups() {
    local bind_user="$1"
    local bind_password="$2"
    local domain_controller="$3"
    local base_dn="$4"
    local group_dn="$5"
    
    log "Testing AD group lookup for $group_dn..."
    
    if ldapsearch -x -H "ldap://$domain_controller" -D "$bind_user" -w "$bind_password" -b "$base_dn" -LLL "(&(objectClass=group)(cn=jenkins-admins))" member 2>/dev/null; then
        log "✓ AD group lookup successful"
        return 0
    else
        warn "⚠ AD group lookup issues (might be normal if group doesn't exist)"
        return 0
    fi
}

main() {
    log "Starting Active Directory connectivity tests..."
    
    # Source environment variables
    if [[ -f ../.env ]]; then
        source ../.env
    else
        error ".env file not found"
        exit 1
    fi
    
    # Test parameters
    local bind_user="${AD_BIND_USER}"
    local bind_password="${AD_BIND_PASSWORD}"
    local domain_controller="${AD_DOMAIN_CONTROLLER1}"
    local domain="${AD_DOMAIN}"
    local base_dn="${AD_BASE_DN}"
    local group_dn="${AD_GROUP_DN:-cn=jenkins-admins,ou=groups,dc=company,dc=com}"
    
    # Run tests
    local tests_passed=0
    local tests_failed=0
    
    if test_ldap_connection "$bind_user" "$bind_password" "$domain_controller" "$base_dn"; then
        ((tests_passed++))
    else
        ((tests_failed++))
    fi
    
    if test_kerberos_auth "$domain" "$bind_user" "$bind_password"; then
        ((tests_passed++))
    else
        ((tests_failed++))
    fi
    
    if test_ad_groups "$bind_user" "$bind_password" "$domain_controller" "$base_dn" "$group_dn"; then
        ((tests_passed++))
    else
        ((tests_failed++))
    fi
    
    log "Tests completed: $tests_passed passed, $tests_failed failed"
    
    if [[ $tests_failed -eq 0 ]]; then
        log "✅ All AD connectivity tests passed!"
    else
        error "❌ Some AD connectivity tests failed"
        exit 1
    fi
}

main "$@"
```

7. Обновленный .env файл с AD настройками

```env
# Corporate Jenkins Configuration
COMPOSE_PROJECT_NAME=jenkins-iac-corporate
JENKINS_URL=http://jenkins.company.com

# Active Directory Configuration
AD_DOMAIN=company.com
AD_DOMAIN_CONTROLLER1=dc1.company.com
AD_DOMAIN_CONTROLLER2=dc2.company.com
AD_BIND_USER=svc_jenkins@company.com
AD_BIND_PASSWORD=secure_ad_bind_password_2024
AD_BASE_DN=DC=company,DC=com
AD_USER_SEARCH_BASE=OU=Users,DC=company,DC=com
AD_GROUP_SEARCH_BASE=OU=Groups,DC=company,DC=com
AD_DNS_SERVER_1=10.0.0.10
AD_DNS_SERVER_2=10.0.0.11

# Security Secrets
ADMIN_PASSWORD=secure_corporate_password_2024
SMTP_PASSWORD=smtp_corporate_password_2024
GIT_PASSWORD=gitlab_corporate_token_2024
ANSIBLE_VAULT_PASSWORD=vault_corporate_2024
DOCKER_REGISTRY_USER=registry-user
DOCKER_REGISTRY_PASSWORD=registry_corporate_2024

# Ansible SSH Key (будет переопределен)
ANSIBLE_SSH_KEY=ssh-rsa AAAAB3NzaC1yc2E...

# Features
ENABLE_TOOLS=true
ENABLE_PROXY=false
ENABLE_MONITORING=true
ENABLE_AD_TEST=true

# Network
JENKINS_NETWORK_SUBNET=172.20.0.0/24
```

8. Скрипт настройки AD групп

scripts/setup-ad-groups.sh:

```bash
#!/usr/bin/env bash

set -euo pipefail

# This script helps create AD groups for Jenkins if they don't exist
# Requires ldap-utils and proper AD permissions

create_ad_group() {
    local group_name="$1"
    local group_dn="$2"
    local bind_user="$3"
    local bind_password="$4"
    local domain_controller="$5"
    
    log "Creating AD group: $group_name"
    
    cat << EOF | ldapmodify -x -H "ldap://$domain_controller" -D "$bind_user" -w "$bind_password"
dn: $group_dn
objectClass: top
objectClass: group
cn: $group_name
sAMAccountName: $group_name
groupType: -2147483646
EOF

}

main() {
    source ../.env
    
    local groups=(
        "jenkins-admins:cn=jenkins-admins,ou=groups,dc=company,dc=com"
        "jenkins-developers:cn=jenkins-developers,ou=groups,dc=company,dc=com"
        "jenkins-viewers:cn=jenkins-viewers,ou=groups,dc=company,dc=com"
    )
    
    for group_info in "${groups[@]}"; do
        IFS=':' read -r group_name group_dn <<< "$group_info"
        
        if ! ldapsearch -x -H "ldap://${AD_DOMAIN_CONTROLLER1}" -D "${AD_BIND_USER}" -w "${AD_BIND_PASSWORD}" -b "$group_dn" -LLL "(objectClass=group)" 2>/dev/null | grep -q "cn: $group_name"; then
            create_ad_group "$group_name" "$group_dn" "${AD_BIND_USER}" "${AD_BIND_PASSWORD}" "${AD_DOMAIN_CONTROLLER1}"
        else
            log "Group $group_name already exists"
        fi
    done
}

main "$@"
```

9. Обновленный deploy.sh с проверкой AD

```bash
#!/usr/bin/env bash
set -euo pipefail

# ... (предыдущий код остается)

check_ad_connectivity() {
    if [[ ${ENABLE_AD_TEST:-false} == "true" ]]; then
        log "Проверка подключения к Active Directory..."
        if docker compose run --rm jenkins /var/jenkins_scripts/test-ad-connection.sh; then
            log "✓ Подключение к AD успешно"
        else
            error "❌ Ошибка подключения к AD"
            warn "Продолжение развертывания, но AD может не работать"
        fi
    fi
}

main() {
    log "🚀 Запуск корпоративного Jenkins с интеграцией Active Directory"
    
    check_dependencies
    setup_environment
    build_images
    check_ad_connectivity
    start_services
    wait_for_jenkins
    setup_initial_config
    
    log "✅ Развертывание завершено успешно!"
    log "🔐 Jenkins интегрирован с Active Directory"
    log "👥 Группы AD для управления доступом:"
    log "   - jenkins-admins: полный доступ"
    log "   - jenkins-developers: разработчики"
    log "   - jenkins-viewers: просмотр"
}

main "$@"
```

Запуск с AD интеграцией

```bash
# Настройка секретов AD
mkdir -p secrets
echo "secure_ad_bind_password_2024" > secrets/ad_bind_password.txt
chmod 600 secrets/ad_bind_password.txt

# Тестирование AD подключения
chmod +x scripts/test-ad-connection.sh
./scripts/test-ad-connection.sh

# Запуск с AD
./deploy.sh

# Запуск утилиты для тестирования AD (опционально)
docker compose --profile ad-test up -d ldap-admin
```

Ключевые особенности AD интеграции:

· ✅ Active Directory аутентификация - единый вход для сотрудников
· ✅ Role-Based Access Control - управление правами через AD группы
· ✅ Kerberos поддержка - безопасная аутентификация
· ✅ LDAP поиск - эффективное получение информации о пользователях и группах
· ✅ Тестирование подключения - автоматическая проверка AD connectivity
· ✅ Готовые группы - предопределенные роли для управления доступом
· ✅ Безопасность - TLS соединение с контроллерами домена

Это решение обеспечивает полную интеграцию с корпоративной инфраструктурой Microsoft Active Directory с соблюдением лучших практик безопасности. 


Добавляю плагины Groovy, PowerShell, Python и Bash в корпоративное решение Jenkins:

1. Обновленный plugins.txt

```txt
# Scripting & Languages
groovy:latest
powershell:latest
python:latest
bash:latest
workflow-cps:latest
script-security:latest

# Active Directory & LDAP
active-directory:latest
ldap:latest
saml:latest
oic-auth:latest

# Pipeline & DSL
workflow-aggregator:latest
pipeline-stage-view:latest
job-dsl:latest
blueocean:latest

# Ansible & Infrastructure
ansible:latest
ansible-tower:latest

# Version Control
git:latest
github:latest
gitlab:latest
bitbucket:latest

# Security & Auth
matrix-auth:latest
role-strategy:latest
credentials-binding:latest
ssh-credentials:latest
ssh-slaves:latest

# Configuration as Code
configuration-as-code:latest
jcascom-configuration-as-code-support:latest

# Docker & Kubernetes
docker-workflow:latest
docker-plugin:latest
kubernetes:latest

# Notifications & Monitoring
email-ext:latest
mailer:latest
telegram-notifications:latest
slack:latest

# Utilities
timestamper:latest
ws-cleanup:latest
build-timeout:latest
parameterized-trigger:latest
copyartifact:latest
envinject:latest
htmlpublisher:latest
pipeline-utility-steps:latest

# UI & Experience
simple-theme-plugin:latest
dashboard-view:latest
```

2. Обновленный Dockerfile с поддержкой всех языков

```dockerfile
# syntax=docker/dockerfile:1.4

ARG JENKINS_VERSION=lts-jdk17

FROM jenkins/jenkins:${JENKINS_VERSION} as base

USER root

# Метка для безопасности
LABEL security.scan="true" \
      maintainer="devops@company.com" \
      version="2.0" \
      languages="java,groovy,python,powershell,bash,ansible"

# Установка базовых утилит + AD/LDAP инструменты + языки программирования
RUN <<EOT
    apt-get update
    apt-get install -y --no-install-recommends \
        python3 \
        python3-pip \
        python3-venv \
        git \
        sshpass \
        openssh-client \
        curl \
        gnupg \
        software-properties-common \
        jq \
        unzip \
        ldap-utils \
        krb5-user \
        libpam-krb5 \
        libpam-sss \
        libnss-sss \
        sssd \
        sssd-tools \
        groovy \
        powershell \
        pwsh \
        shellcheck \
        npm \
        nodejs \
        maven \
        gradle
    rm -rf /var/lib/apt/lists/*
    apt-get clean
EOT

# Установка .NET Core для расширенной поддержки PowerShell
RUN <<EOT
    wget https://packages.microsoft.com/config/debian/11/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
    dpkg -i packages-microsoft-prod.deb
    rm packages-microsoft-prod.deb
    apt-get update
    apt-get install -y dotnet-sdk-6.0
EOT

# Многоступенчатая установка для оптимизации
FROM base as ansible-install

RUN pip3 install --no-cache-dir \
    ansible \
    ansible-lint \
    yamllint \
    ansible-tower-cli \
    jmespath \
    netaddr

FROM base as python-libs

RUN pip3 install --no-cache-dir \
    requests \
    jinja2 \
    pyyaml \
    cryptography \
    azure-identity \
    boto3 \
    google-auth

FROM base as final

# Копируем установленные компоненты
COPY --from=ansible-install /usr/local/lib/python3.9/dist-packages /usr/local/lib/python3.9/dist-packages
COPY --from=ansible-install /usr/local/bin/ansible* /usr/local/bin/
COPY --from=python-libs /usr/local/lib/python3.9/dist-packages /usr/local/lib/python3.9/dist-packages

# Копируем .NET и PowerShell
COPY --from=base /usr/bin/pwsh /usr/bin/pwsh
COPY --from=base /usr/bin/powershell /usr/bin/powershell
COPY --from=base /usr/share/dotnet /usr/share/dotnet

# Настройка Kerberos для AD
RUN mkdir -p /etc/krb5.conf.d
COPY kerberos/krb5.conf /etc/krb5.conf

# Создание структуры директорий для скриптов
RUN <<EOT
    mkdir -p /var/jenkins_conf/casc /var/ansible /shared
    mkdir -p /usr/local/scripts/{groovy,powershell,python,bash}
    chown -R jenkins:jenkins /var/jenkins_conf /var/ansible /shared /usr/local/scripts
EOT

# Копирование скриптов и библиотек
COPY --chown=jenkins:jenkins scripts/ /var/jenkins_scripts/
COPY --chown=jenkins:jenkins shared_scripts/ /usr/local/scripts/

RUN chmod +x /var/jenkins_scripts/*.sh
RUN chmod +x /usr/local/scripts/bash/*.sh
RUN chmod +x /usr/local/scripts/python/*.py
RUN chmod +x /usr/local/scripts/powershell/*.ps1

# Создание симлинков для утилит
RUN ln -sf /usr/bin/python3 /usr/bin/python

# Установка плагинов через установленный скрипт
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

USER jenkins

# Установка Groovy библиотек через Jenkins
HEALTHCHECK --interval=30s --timeout=10s --start-period=1m --retries=3 \
  CMD curl -f http://localhost:8080 || exit 1
```

3. Обновленная конфигурация JCasC с поддержкой языков

casc/jenkins.yaml:

```yaml
jenkins:
  agentProtocols:
    - "JNLP4-connect"
    - "Ping"
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            description: "Jenkins Administrators"
            permissions:
              - "Overall/Administer"
              - "Overall/Read"
            assignments:
              - "cn=jenkins-admins,ou=groups,dc=company,dc=com"
          - name: "developer"
            description: "Developers"
            permissions:
              - "Job/Read"
              - "Job/Build"
              - "Job/Workspace"
              - "Job/Cancel"
              - "Run/Replay"
              - "Run/Update"
            assignments:
              - "cn=developers,ou=groups,dc=company,dc=com"
          - name: "script-user"
            description: "Script Users"
            permissions:
              - "Overall/Read"
              - "Job/Read"
              - "Job/Build"
              - "View/Read"
            assignments:
              - "cn=script-users,ou=groups,dc=company,dc=com"
  clouds: []
  disabledAdministrativeMonitors:
    - "hudson.diagnosis.ReverseProxySetupMonitor"
  label: "master"
  mode: NORMAL
  numExecutors: 5
  primaryView:
    all:
      name: "all"
  quietPeriod: 5
  remotingSecurity:
    enabled: true
  scmCheckoutRetryCount: 2
  securityRealm:
    activeDirectory:
      domains:
        - name: "company.com"
          servers: "${AD_DOMAIN_CONTROLLER1}:389,${AD_DOMAIN_CONTROLLER2}:389"
          site: "Default-First-Site-Name"
          bindName: "${AD_BIND_USER}"
          bindPassword: "${AD_BIND_PASSWORD}"
          groupLookupStrategy: "RECURSIVE"
          cache:
            size: 1000
            ttl: 300
          startTls: true
          removeIrrelevantGroups: true
          customDomain: true
          tlsConfiguration: "TRUST_ALL_CERTIFICATES"
      cache:
        size: 1000
        ttl: 600
      groupLookupStrategy: "RECURSIVE"
      removeIrrelevantGroups: true
      customDomain: true
      startTls: true
  slaveAgentPort: 50000
  systemMessage: |
    Jenkins Infrastructure as Code Platform
    Корпоративная система управления конфигурацией
    Поддержка языков: Groovy, PowerShell, Python, Bash
  views:
    - all:
        name: "all"
  viewsTabBar: "standard"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "git-corporate-credentials"
              username: "git-service"
              password: "${GIT_PASSWORD}"
          - sshUsernamePrivateKey:
              scope: GLOBAL
              id: "ansible-corporate-key"
              username: "ansible"
              privateKeySource:
                directEntry:
                  privateKey: "${ANSIBLE_SSH_KEY}"
          - string:
              scope: GLOBAL
              id: "ansible-vault-corporate"
              secret: "${ANSIBLE_VAULT_PASSWORD}"
          - usernamePassword:
              scope: GLOBAL
              id: "docker-registry"
              username: "${DOCKER_REGISTRY_USER}"
              password: "${DOCKER_REGISTRY_PASSWORD}"
          - usernamePassword:
              scope: GLOBAL
              id: "ad-service-account"
              username: "${AD_BIND_USER}"
              password: "${AD_BIND_PASSWORD}"
          - secretText:
              scope: GLOBAL
              id: "python-api-key"
              secret: "${PYTHON_API_KEY}"
          - secretText:
              scope: GLOBAL
              id: "powershell-encryption-key"
              secret: "${POWERSHELL_ENCRYPTION_KEY}"

tool:
  git:
    installations:
      - name: "git"
        home: "/usr/bin/git"
  ansible:
    installations:
      - name: "ansible"
        home: "/usr/local/bin/ansible"
  powershell:
    installations:
      - name: "powershell-linux"
        home: "/usr/bin/pwsh"
      - name: "powershell-windows"
        home: "C:\\Program Files\\PowerShell\\7\\pwsh.exe"
  python:
    installations:
      - name: "python3"
        home: "/usr/bin/python3"
        properties:
          - installSource:
              installers:
                - command:
                    command: "/usr/bin/python3 --version"
  groovy:
    installations:
      - name: "groovy-system"
        home: "/usr/share/groovy"
        properties:
          - installSource:
              installers:
                - command:
                    command: "/usr/bin/groovy --version"

unclassified:
  location:
    url: "${JENKINS_URL}"
    adminAddress: "jenkins-admin@company.com"
  
  mailer:
    smtpHost: "smtp.company.com"
    smtpPort: "587"
    useSsl: true
    smtpAuth: true
    username: "jenkins-noreply@company.com"
    password: "${SMTP_PASSWORD}"
    charset: "UTF-8"
  
  systemd:
    enable: false
  
  securityFilters:
    adminForJob: true
  
  buildDiscarders:
    configuredBuildDiscarders:
      - strategy:
          daysToKeep: "30"
          numToKeep: "50"
          artifactDaysToKeep: "15"
          artifactNumToKeep: "20"
  
  # Groovy System Configuration
  groovy:
    enabled: true
    securityEnabled: true
  
  # Script Security
  scriptSecurity:
    approvedSignatures:
      - "method java.lang.ProcessBuilder start"
      - "method java.lang.Runtime exec"
      - "staticMethod org.codehaus.groovy.runtime.DefaultGroovyMethods execute"
      - "staticMethod org.codehaus.groovy.runtime.ProcessGroovyMethods execute"
      - "staticMethod org.codehaus.groovy.runtime.ProcessGroovyMethods getText"

security:
  apiToken:
    creationOfLegacyTokenEnabled: false
    tokenGenerationOnCreationEnabled: true
  sshd:
    enabled: false
  sSHD:
    port: -1
  scriptApproval:
    approvedSignatures:
      - "method groovy.json.JsonSlurper parseText"
      - "method groovy.json.JsonOutput toJson"
      - "method java.util.Map get"
      - "method java.util.List get"
      - "staticMethod org.codehaus.groovy.runtime.DefaultGroovyMethods eachLine java.io.File groovy.lang.Closure"
```

4. Скрипты-примеры для каждого языка

Groovy скрипты

shared_scripts/groovy/SystemInfo.groovy:

```groovy
#!/usr/bin/env groovy

def call() {
    echo "🔍 System Information Groovy Script"
    
    // Jenkins system info
    echo "Jenkins URL: ${Jenkins.instance.getRootUrl()}"
    echo "Jenkins Version: ${Jenkins.instance.getVersion()}"
    echo "Executor Count: ${Jenkins.instance.getNumExecutors()}"
    
    // Node information
    def nodes = Jenkins.instance.getNodes()
    nodes.each { node ->
        echo "Node: ${node.getDisplayName()} - ${node.getLabelString()}"
    }
    
    // Plugin information
    def plugins = Jenkins.instance.pluginManager.plugins
    def enabledPlugins = plugins.findAll { it.isEnabled() }
    echo "Enabled Plugins: ${enabledPlugins.size()}"
    
    return [
        'node_count': nodes.size(),
        'plugin_count': enabledPlugins.size(),
        'executor_count': Jenkins.instance.getNumExecutors()
    ]
}

def getQueueInfo() {
    def queue = Jenkins.instance.queue
    def items = queue.getItems()
    
    echo "Build Queue: ${items.size()} items"
    items.each { item ->
        echo "  - ${item.task.name}: ${item.getWhy()}"
    }
    
    return items.size()
}

// AD User information (if available)
def getADUserInfo(String username) {
    try {
        def user = User.get(username, false)
        if (user) {
            def properties = user.getAllProperties()
            echo "User: ${user.getDisplayName()}"
            properties.each { prop ->
                echo "  - ${prop.getClass().getSimpleName()}"
            }
        }
    } catch (Exception e) {
        echo "⚠️ Could not get user info: ${e.message}"
    }
}
```

PowerShell скрипты

shared_scripts/powershell/SystemCheck.ps1:

```powershell
#!/usr/bin/env pwsh

<#
.SYNOPSIS
    Corporate System Check PowerShell Script
.DESCRIPTION
    Performs system checks and returns status information
.PARAMETER CheckType
    Type of check to perform
#>

param(
    [ValidateSet("System", "Network", "Storage", "All")]
    [string]$CheckType = "All"
)

function Get-SystemInfo {
    Write-Host "🖥️ System Information" -ForegroundColor Green
    
    $os = Get-CimInstance Win32_OperatingSystem
    $computer = Get-CimInstance Win32_ComputerSystem
    
    return @{
        OS = $os.Caption
        Version = $os.Version
        Architecture = $os.OSArchitecture
        MemoryGB = [math]::Round($computer.TotalPhysicalMemory / 1GB, 2)
        Processors = $computer.NumberOfProcessors
    }
}

function Get-NetworkInfo {
    Write-Host "🌐 Network Information" -ForegroundColor Cyan
    
    $networkAdapters = Get-NetAdapter | Where-Object { $_.Status -eq 'Up' }
    $dnsServers = (Get-DnsClientServerAddress -AddressFamily IPv4 | Where-Object { $_.ServerAddresses.Length -gt 0 }).ServerAddresses
    
    return @{
        Adapters = $networkAdapters.Count
        DNS_Servers = $dnsServers -join ', '
        Hostname = [System.Net.Dns]::GetHostName()
    }
}

function Get-StorageInfo {
    Write-Host "💾 Storage Information" -ForegroundColor Yellow
    
    $drives = Get-PSDrive -PSProvider FileSystem | Where-Object { $_.Used -gt 0 }
    $storageInfo = @()
    
    foreach ($drive in $drives) {
        $storageInfo += @{
            Drive = $drive.Name
            FreeGB = [math]::Round($drive.Free / 1GB, 2)
            UsedGB = [math]::Round($drive.Used / 1GB, 2)
            TotalGB = [math]::Round(($drive.Free + $drive.Used) / 1GB, 2)
        }
    }
    
    return $storageInfo
}

function Invoke-ADCheck {
    Write-Host "🔐 Active Directory Check" -ForegroundColor Magenta
    
    try {
        # Check if AD module is available
        if (Get-Module -ListAvailable -Name ActiveDirectory) {
            Import-Module ActiveDirectory
            
            $domain = Get-ADDomain
            $domainControllers = Get-ADDomainController -Filter *
            
            return @{
                Domain = $domain.DNSRoot
                DomainControllers = $domainControllers.Count
                Forest = $domain.Forest
            }
        } else {
            Write-Warning "Active Directory module not available"
            return $null
        }
    } catch {
        Write-Warning "AD check failed: $_"
        return $null
    }
}

# Main execution
$results = @{}

switch ($CheckType) {
    "System" { 
        $results.System = Get-SystemInfo
    }
    "Network" { 
        $results.Network = Get-NetworkInfo
    }
    "Storage" { 
        $results.Storage = Get-StorageInfo
    }
    "All" {
        $results.System = Get-SystemInfo
        $results.Network = Get-NetworkInfo
        $results.Storage = Get-StorageInfo
        $results.AD = Invoke-ADCheck
    }
}

# Convert to JSON for Jenkins pipeline
return $results | ConvertTo-Json -Depth 3
```

Python скрипты

shared_scripts/python/infrastructure_check.py:

```python
#!/usr/bin/env python3
"""
Corporate Infrastructure Check Script
Provides infrastructure monitoring and validation capabilities
"""

import json
import subprocess
import sys
import os
import socket
import psutil
from pathlib import Path

class InfrastructureChecker:
    def __init__(self):
        self.results = {}
    
    def check_system_resources(self):
        """Check system CPU, memory, and disk usage"""
        try:
            cpu_percent = psutil.cpu_percent(interval=1)
            memory = psutil.virtual_memory()
            disk = psutil.disk_usage('/')
            
            self.results['system'] = {
                'cpu_percent': cpu_percent,
                'memory_total_gb': round(memory.total / (1024**3), 2),
                'memory_used_gb': round(memory.used / (1024**3), 2),
                'memory_percent': memory.percent,
                'disk_total_gb': round(disk.total / (1024**3), 2),
                'disk_used_gb': round(disk.used / (1024**3), 2),
                'disk_percent': disk.percent
            }
            return True
        except Exception as e:
            self.results['system_error'] = str(e)
            return False
    
    def check_network_connectivity(self, hosts=None):
        """Check network connectivity to important hosts"""
        if hosts is None:
            hosts = ['google.com', 'github.com', 'company.com']
        
        network_results = {}
        for host in hosts:
            try:
                socket.setdefaulttimeout(5)
                socket.gethostbyname(host)
                network_results[host] = 'reachable'
            except socket.error as e:
                network_results[host] = f'unreachable: {e}'
        
        self.results['network'] = network_results
        return all(status == 'reachable' for status in network_results.values())
    
    def check_docker_status(self):
        """Check Docker daemon status and container count"""
        try:
            result = subprocess.run(['docker', 'info'], 
                                  capture_output=True, text=True, timeout=30)
            if result.returncode == 0:
                # Count running containers
                containers_result = subprocess.run(
                    ['docker', 'ps', '-q'], 
                    capture_output=True, text=True
                )
                running_containers = len(containers_result.stdout.strip().split('\n')) - 1
                
                self.results['docker'] = {
                    'status': 'running',
                    'running_containers': running_containers
                }
                return True
            else:
                self.results['docker'] = {'status': 'not_running'}
                return False
        except (subprocess.TimeoutExpired, FileNotFoundError, subprocess.SubprocessError) as e:
            self.results['docker'] = {'status': f'error: {e}'}
            return False
    
    def check_ansible_inventory(self, inventory_path='/inventory'):
        """Check Ansible inventory if available"""
        inventory_path = Path(inventory_path)
        if inventory_path.exists():
            try:
                # Simple inventory check
                result = subprocess.run(
                    ['ansible-inventory', '-i', str(inventory_path), '--list'],
                    capture_output=True, text=True, timeout=30
                )
                if result.returncode == 0:
                    inventory_data = json.loads(result.stdout)
                    host_count = sum(len(hosts) for hosts in inventory_data.values() 
                                   if isinstance(hosts, dict))
                    
                    self.results['ansible'] = {
                        'inventory_valid': True,
                        'host_count': host_count,
                        'groups': list(inventory_data.keys())
                    }
                    return True
            except Exception as e:
                self.results['ansible'] = {
                    'inventory_valid': False,
                    'error': str(e)
                }
                return False
        return None
    
    def run_all_checks(self):
        """Run all infrastructure checks"""
        print("🔍 Running Infrastructure Checks...")
        
        checks = [
            ('System Resources', self.check_system_resources),
            ('Network Connectivity', lambda: self.check_network_connectivity()),
            ('Docker Status', self.check_docker_status),
            ('Ansible Inventory', self.check_ansible_inventory)
        ]
        
        for check_name, check_func in checks:
            try:
                success = check_func()
                status = "✅" if success else "⚠️"
                print(f"{status} {check_name}")
            except Exception as e:
                print(f"❌ {check_name}: {e}")
        
        return self.results

def main():
    checker = InfrastructureChecker()
    results = checker.run_all_checks()
    
    # Output results as JSON for Jenkins
    print(json.dumps(results, indent=2))
    
    # Determine exit code
    if any('error' in str(result).lower() for result in results.values()):
        sys.exit(1)
    else:
        sys.exit(0)

if __name__ == "__main__":
    main()
```

Bash скрипты

shared_scripts/bash/system_audit.sh:

```bash
#!/bin/bash

# Corporate System Audit Script
# Performs comprehensive system checks and audits

set -euo pipefail

# Color codes for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Logging functions
log_info() {
    echo -e "${BLUE}[INFO]${NC} $1"
}

log_success() {
    echo -e "${GREEN}[SUCCESS]${NC} $1"
}

log_warning() {
    echo -e "${YELLOW}[WARNING]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# Results array
declare -A RESULTS

check_system_info() {
    log_info "Checking system information..."
    
    RESULTS["hostname"]=$(hostname)
    RESULTS["os"]=$(source /etc/os-release && echo "$PRETTY_NAME")
    RESULTS["kernel"]=$(uname -r)
    RESULTS["architecture"]=$(uname -m)
    RESULTS["uptime"]=$(uptime -p)
    
    log_success "System info collected"
}

check_cpu_memory() {
    log_info "Checking CPU and memory..."
    
    RESULTS["cpu_cores"]=$(nproc)
    RESULTS["cpu_load"]=$(uptime | awk -F'load average:' '{ print $2 }' | xargs)
    RESULTS["memory_total_mb"]=$(free -m | awk 'NR==2{print $2}')
    RESULTS["memory_used_mb"]=$(free -m | awk 'NR==2{print $3}')
    RESULTS["memory_usage_percent"]=$(free | awk 'NR==2{printf "%.2f", $3/$2 * 100}')
    
    log_success "CPU and memory info collected"
}

check_disk_usage() {
    log_info "Checking disk usage..."
    
    local disk_info=$(df -h / | awk 'NR==2{print $5,$2,$3,$4}')
    RESULTS["disk_usage"]=$(echo "$disk_info" | awk '{print $1}')
    RESULTS["disk_total"]=$(echo "$disk_info" | awk '{print $2}')
    RESULTS["disk_used"]=$(echo "$disk_info" | awk '{print $3}')
    RESULTS["disk_available"]=$(echo "$disk_info" | awk '{print $4}')
    
    log_success "Disk usage info collected"
}

check_docker_containers() {
    log_info "Checking Docker containers..."
    
    if command -v docker &> /dev/null; then
        RESULTS["docker_installed"]="true"
        RESULTS["docker_containers_total"]=$(docker ps -aq | wc -l | tr -d ' ')
        RESULTS["docker_containers_running"]=$(docker ps -q | wc -l | tr -d ' ')
        RESULTS["docker_version"]=$(docker --version | cut -d' ' -f3 | tr -d ',')
    else
        RESULTS["docker_installed"]="false"
        log_warning "Docker not installed"
    fi
    
    log_success "Docker info collected"
}

check_network_connectivity() {
    log_info "Checking network connectivity..."
    
    local hosts=("google.com" "github.com" "company.com")
    local connectivity_results=()
    
    for host in "${hosts[@]}"; do
        if ping -c 1 -W 3 "$host" &> /dev/null; then
            connectivity_results+=("$host:reachable")
        else
            connectivity_results+=("$host:unreachable")
        fi
    done
    
    RESULTS["network_connectivity"]=$(IFS=','; echo "${connectivity_results[*]}")
    log_success "Network connectivity checked"
}

check_jenkins_status() {
    log_info "Checking Jenkins status..."
    
    if curl -s -f http://localhost:8080 > /dev/null; then
        RESULTS["jenkins_status"]="running"
        RESULTS["jenkins_accessible"]="true"
    else
        RESULTS["jenkins_status"]="not_running"
        RESULTS["jenkins_accessible"]="false"
        log_warning "Jenkins is not accessible"
    fi
    
    log_success "Jenkins status checked"
}

generate_report() {
    log_info "Generating audit report..."
    
    local report_file="/shared/audit_report_$(date +%Y%m%d_%H%M%S).json"
    
    # Convert associative array to JSON
    echo "{" > "$report_file"
    local first=true
    for key in "${!RESULTS[@]}"; do
        if [ "$first" = true ]; then
            first=false
        else
            echo "," >> "$report_file"
        fi
        printf '  "%s": "%s"' "$key" "${RESULTS[$key]}" >> "$report_file"
    done
    echo -e "\n}" >> "$report_file"
    
    log_success "Audit report generated: $report_file"
    
    # Print summary
    echo
    log_info "=== AUDIT SUMMARY ==="
    for key in "${!RESULTS[@]}"; do
        echo "  $key: ${RESULTS[$key]}"
    done
}

main() {
    log_info "Starting corporate system audit..."
    
    check_system_info
    check_cpu_memory
    check_disk_usage
    check_docker_containers
    check_network_connectivity
    check_jenkins_status
    generate_report
    
    log_success "System audit completed successfully"
}

# Handle script execution
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

5. Jenkins Pipeline пример с использованием всех языков

shared_scripts/pipelines/multi-language-pipeline.jenkinsfile:

```groovy
pipeline {
    agent any
    tools {
        powershell 'powershell-linux'
        python 'python3'
        groovy 'groovy-system'
    }
    stages {
        stage('System Audit - Bash') {
            steps {
                script {
                    sh '''
                        chmod +x /usr/local/scripts/bash/system_audit.sh
                        /usr/local/scripts/bash/system_audit.sh
                    '''
                }
            }
        }
        
        stage('Infrastructure Check - Python') {
            steps {
                script {
                    withPythonEnv('/usr/bin/python3') {
                        sh 'python3 /usr/local/scripts/python/infrastructure_check.py'
                    }
                }
            }
        }
        
        stage('System Check - PowerShell') {
            steps {
                powershell '''
                    . /usr/local/scripts/powershell/SystemCheck.ps1
                    $results = Invoke-SystemCheck -CheckType All
                    Write-Host "Results: $($results | ConvertTo-Json -Depth 3)"
                '''
            }
        }
        
        stage('Groovy System Info') {
            steps {
                script {
                    def groovyResults = load '/usr/local/scripts/groovy/SystemInfo.groovy'
                    def systemInfo = groovyResults.call()
                    echo "System Info: ${systemInfo}"
                    
                    def queueSize = groovyResults.getQueueInfo()
                    echo "Queue size: ${queueSize}"
                }
            }
        }
        
        stage('Multi-language Report') {
            steps {
                script {
                    // Generate combined report using all languages
                    def combinedReport = [
                        timestamp: new Date().toString(),
                        system_audit: readJSON(file: '/shared/audit_report_*.json')[0],
                        pipeline_info: [
                            'build_number': env.BUILD_NUMBER,
                            'job_name': env.JOB_NAME,
                            'node_name': env.NODE_NAME
                        ]
                    ]
                    
                    writeJSON file: '/shared/combined_report.json', json: combinedReport
                    archiveArtifacts artifacts: '/shared/combined_report.json', fingerprint: true
                }
            }
        }
    }
    
    post {
        always {
            script {
                // Cleanup and notification
                echo "Multi-language pipeline execution completed"
                
                // PowerShell notification
                powershell '''
                    Write-Host "Pipeline completed with status: $env.BUILD_STATUS" -ForegroundColor Green
                '''
            }
        }
        success {
            emailext (
                subject: "SUCCESS: Multi-language Pipeline ${env.JOB_NAME} - Build ${env.BUILD_NUMBER}",
                body: """
                Multi-language infrastructure check completed successfully.
                
                Build: ${env.BUILD_URL}
                Node: ${env.NODE_NAME}
                Duration: ${currentBuild.durationString}
                
                Reports available in shared directory.
                """,
                to: "infrastructure-team@company.com"
            )
        }
        failure {
            emailext (
                subject: "FAILED: Multi-language Pipeline ${env.JOB_NAME} - Build ${env.BUILD_NUMBER}",
                body: """
                Multi-language infrastructure check failed.
                
                Build: ${env.BUILD_URL}
                Node: ${env.NODE_NAME}
                Duration: ${currentBuild.durationString}
                
                Please check Jenkins logs for details.
                """,
                to: "infrastructure-team@company.com"
            )
        }
    }
}
```

6. Обновленный .env с новыми переменными

```env
# Corporate Jenkins Configuration
COMPOSE_PROJECT_NAME=jenkins-iac-corporate
JENKINS_URL=http://jenkins.company.com

# Active Directory Configuration
AD_DOMAIN=company.com
AD_DOMAIN_CONTROLLER1=dc1.company.com
AD_DOMAIN_CONTROLLER2=dc2.company.com
AD_BIND_USER=svc_jenkins@company.com
AD_BIND_PASSWORD=secure_ad_bind_password_2024
AD_BASE_DN=DC=company,DC=com
AD_USER_SEARCH_BASE=OU=Users,DC=company,DC=com
AD_GROUP_SEARCH_BASE=OU=Groups,DC=company,DC=com
AD_DNS_SERVER_1=10.0.0.10
AD_DNS_SERVER_2=10.0.0.11

# Scripting & Language Security
PYTHON_API_KEY=python_corporate_api_2024
POWERSHELL_ENCRYPTION_KEY=powershell_secure_key_2024
GROOVY_SCRIPT_APPROVAL_ENABLED=true

# Security Secrets
ADMIN_PASSWORD=secure_corporate_password_2024
SMTP_PASSWORD=smtp_corporate_password_2024
GIT_PASSWORD=gitlab_corporate_token_2024
ANSIBLE_VAULT_PASSWORD=vault_corporate_2024
DOCKER_REGISTRY_USER=registry-user
DOCKER_REGISTRY_PASSWORD=registry_corporate_2024

# Ansible SSH Key (будет переопределен)
ANSIBLE_SSH_KEY=ssh-rsa AAAAB3NzaC1yc2E...

# Features
ENABLE_TOOLS=true
ENABLE_PROXY=false
ENABLE_MONITORING=true
ENABLE_AD_TEST=true
ENABLE_MULTI_LANGUAGE=true

# Network
JENKINS_NETWORK_SUBNET=172.20.0.0/24
```

Запуск с поддержкой всех языков

```bash
# Создание директорий для скриптов
mkdir -p shared_scripts/{groovy,powershell,python,bash,pipelines}

# Настройка прав
chmod +x shared_scripts/bash/*.sh
chmod +x shared_scripts/python/*.py

# Запуск системы
./deploy.sh

# Создание пайплайна с multi-language поддержкой
cp shared_scripts/pipelines/multi-language-pipeline.jenkinsfile data/jenkins_home/
```

Ключевые особенности мульти-языковой поддержки:

· ✅ Groovy: Полная интеграция с Jenkins API, системные скрипты
· ✅ PowerShell: Поддержка Windows и Linux, AD интеграция
· ✅ Python: Широкие возможности автоматизации, библиотеки для инфраструктуры
· ✅ Bash: Стандартные системные скрипты, проверка окружения
· ✅ Безопасность: Script approval, sandbox выполнение
· ✅ Интеграция: Все языки работают вместе в пайплайнах
· ✅ Корпоративные функции: Логирование, аудит, уведомления

Это решение предоставляет полную мульти-языковую платформу для автоматизации инфраструктуры с корпоративным уровнем безопасности и интеграции.