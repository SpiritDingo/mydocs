Я создам для вас Dockerfile и docker-compose.yml для Jenkins с указанными плагинами.

Dockerfile

```dockerfile
FROM jenkins/jenkins:lts-jdk17

# Переключаемся на root для установки пакетов
USER root

# Устанавливаем необходимые пакеты
RUN apt-get update && apt-get install -y \
    curl \
    gnupg \
    software-properties-common \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Устанавливаем Ansible
RUN pip3 install ansible

# Возвращаемся к пользователю jenkins
USER jenkins

# Устанавливаем плагины Jenkins
RUN jenkins-plugin-cli --plugins \
    ansible:1.1 \
    gitlab-plugin:1.7.16 \
    blueocean:1.27.6 \
    git:5.0.0 \
    workflow-aggregator:2.6 \
    credentials-binding:1.27 \
    docker-workflow:1.29 \
    pipeline-utility-steps:2.15.0 \
    ssh-slaves:1.32.0 \
    matrix-auth:3.1.6

# Устанавливаем переменные окружения
ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"
ENV JENKINS_HOME="/var/jenkins_home"

# Копируем конфигурационные файлы
COPY --chown=jenkins:jenkins config/ /var/jenkins_home/

# Открываем порты
EXPOSE 8080 50000
```

docker-compose.yml

```yaml
version: '3.8'

services:
  jenkins:
    build: 
      context: .
      dockerfile: Dockerfile
    container_name: jenkins-with-plugins
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
    environment:
      - JENKINS_OPTS=--prefix=/jenkins
    networks:
      - jenkins-network

volumes:
  jenkins_home:
    driver: local

networks:
  jenkins-network:
    driver: bridge
```

Дополнительные файлы конфигурации

Создайте директорию config и файл config/init.groovy для начальной настройки:

```groovy
// config/init.groovy
import jenkins.model.*
import hudson.security.*
import jenkins.security.s2m.AdminWhitelistRule

def instance = Jenkins.getInstance()

// Создаем пользователя администратора
def hudsonRealm = new HudsonPrivateSecurityRealm(false)
hudsonRealm.createAccount("admin", "admin123")
instance.setSecurityRealm(hudsonRealm)

// Настраиваем авторизацию
def strategy = new FullControlOnceLoggedInAuthorizationStrategy()
strategy.setAllowAnonymousRead(false)
instance.setAuthorizationStrategy(strategy)

// Отключаем мастер-агент security
instance.getInjector().getInstance(AdminWhitelistRule.class).setMasterKillSwitch(false)

// Сохраняем настройки
instance.save()
```

Файл для установки дополнительных инструментов (опционально)

Создайте файл install-tools.sh если нужно установить дополнительные утилиты:

```bash
#!/bin/bash

# Установка дополнительных инструментов
apt-get update
apt-get install -y \
    sshpass \
    rsync \
    jq \
    unzip

# Очистка кеша
apt-get clean
rm -rf /var/lib/apt/lists/*
```

Инструкции по запуску

1. Создайте структуру файлов:

```
jenkins-docker/
├── Dockerfile
├── docker-compose.yml
└── config/
    └── init.groovy
```

1. Соберите и запустите контейнер:

```bash
docker-compose up --build -d
```

1. Откройте Jenkins в браузере:

```
http://localhost:8080
```

1. Войдите с учетными данными:

· Логин: admin
· Пароль: admin123

Особенности данной конфигурации:

· Ansible: Установлен через pip3 в контейнере
· GitLab Plugin: Для интеграции с GitLab
· Blue Ocean: Современный UI для Jenkins pipelines
· Docker socket: Монтируется для использования Docker внутри контейнера
· Persistent storage: Данные сохраняются в volume jenkins_home
· Автоматическая настройка: Groovy скрипт создает администратора

Вы можете настроить пароли и дополнительные плагины по своему усмотрению.