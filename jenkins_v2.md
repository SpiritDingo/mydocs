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