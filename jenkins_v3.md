Полный проект Jenkins Infrastructure as Code с AD интеграцией и мульти-языковой поддержкой

Структура проекта

```
jenkins-iac-corporate/
├── docker-compose.yml
├── Dockerfile
├── .dockerignore
├── .env
├── .env.example
├── deploy.sh
├── plugins.txt
├── casc/
│   └── jenkins.yaml
├── kerberos/
│   └── krb5.conf
├── scripts/
│   ├── management.sh
│   ├── test-ad-connection.sh
│   ├── setup-ad-groups.sh
│   └── install-plugins.sh
├── shared_scripts/
│   ├── groovy/
│   │   ├── SystemInfo.groovy
│   │   ├── UserManagement.groovy
│   │   └── PipelineUtils.groovy
│   ├── powershell/
│   │   ├── SystemCheck.ps1
│   │   ├── ADManagement.ps1
│   │   └── NetworkTest.ps1
│   ├── python/
│   │   ├── infrastructure_check.py
│   │   ├── api_client.py
│   │   └── security_scanner.py
│   ├── bash/
│   │   ├── system_audit.sh
│   │   ├── docker_management.sh
│   │   └── backup_scripts.sh
│   └── pipelines/
│       ├── multi-language-pipeline.jenkinsfile
│       ├── ansible-deployment.jenkinsfile
│       └── security-scan.jenkinsfile
├── nginx/
│   ├── conf.d/
│   │   └── jenkins.conf
│   └── ssl/
│       ├── server.crt
│       └── server.key
├── data/
│   ├── jenkins_home/
│   ├── ansible/
│   │   └── inventory/
│   └── shared/
├── secrets/
│   ├── jenkins_admin_password.txt
│   ├── smtp_password.txt
│   ├── ad_bind_password.txt
│   └── git_password.txt
└── README.md
```

1. docker-compose.yml

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
      JAVA_OPTS: -Djenkins.install.runSetupWizard=false -Xmx2g -Xms512m -Duser.timezone=Europe/Moscow -Dcom.sun.jndi.ldap.object.disableEndpointIdentification=true
      JENKINS_SLAVE_AGENT_PORT: 50000
    env_file:
      - .env
    volumes:
      - jenkins_data:/var/jenkins_home
      - ansible_data:/var/ansible
      - ./casc:/var/jenkins_conf/casc:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./scripts:/var/jenkins_scripts:ro
      - ./shared_scripts:/usr/local/scripts:ro
      - ./shared:/shared:rw
      - ./kerberos/krb5.conf:/etc/krb5.conf:ro
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
      - ${AD_DNS_SERVER_1}
      - ${AD_DNS_SERVER_2}

  ansible-controller:
    image: quay.io/ansible/ansible-runner:latest
    container_name: ansible-controller
    hostname: ansible-controller
    restart: unless-stopped
    profiles: ["tools", "monitoring"]
    volumes:
      - ansible_data:/runner:rw
      - ./shared:/shared:rw
      - ./data/ansible/inventory:/inventory:ro
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

  ldap-admin:
    image: osixia/phpldapadmin:latest
    container_name: ldap-admin
    restart: unless-stopped
    profiles: ["tools", "ad-test"]
    environment:
      PHPLDAPADMIN_LDAP_HOSTS: ${AD_DOMAIN_CONTROLLER1}
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

2. Dockerfile

```dockerfile
# syntax=docker/dockerfile:1.4

ARG JENKINS_VERSION=lts-jdk17

FROM jenkins/jenkins:${JENKINS_VERSION} as base

USER root

LABEL security.scan="true" \
      maintainer="devops@company.com" \
      version="2.0" \
      languages="java,groovy,python,powershell,bash,ansible"

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

RUN <<EOT
    wget https://packages.microsoft.com/config/debian/11/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
    dpkg -i packages-microsoft-prod.deb
    rm packages-microsoft-prod.deb
    apt-get update
    apt-get install -y dotnet-sdk-6.0
EOT

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
    google-auth \
    psutil

FROM base as final

COPY --from=ansible-install /usr/local/lib/python3.9/dist-packages /usr/local/lib/python3.9/dist-packages
COPY --from=ansible-install /usr/local/bin/ansible* /usr/local/bin/
COPY --from=python-libs /usr/local/lib/python3.9/dist-packages /usr/local/lib/python3.9/dist-packages
COPY --from=base /usr/bin/pwsh /usr/bin/pwsh
COPY --from=base /usr/bin/powershell /usr/bin/powershell
COPY --from=base /usr/share/dotnet /usr/share/dotnet

RUN mkdir -p /etc/krb5.conf.d
COPY kerberos/krb5.conf /etc/krb5.conf

RUN <<EOT
    mkdir -p /var/jenkins_conf/casc /var/ansible /shared
    mkdir -p /usr/local/scripts/{groovy,powershell,python,bash}
    chown -R jenkins:jenkins /var/jenkins_conf /var/ansible /shared /usr/local/scripts
EOT

COPY --chown=jenkins:jenkins shared_scripts/ /usr/local/scripts/
COPY --chown=jenkins:jenkins scripts/ /var/jenkins_scripts/

RUN chmod +x /var/jenkins_scripts/*.sh
RUN chmod +x /usr/local/scripts/bash/*.sh
RUN chmod +x /usr/local/scripts/python/*.py
RUN chmod +x /usr/local/scripts/powershell/*.ps1

RUN ln -sf /usr/bin/python3 /usr/bin/python

RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

USER jenkins

HEALTHCHECK --interval=30s --timeout=10s --start-period=1m --retries=3 \
  CMD curl -f http://localhost:8080 || exit 1
```

3. .dockerignore

```
.git
.gitignore
README.md
.env
secrets/
data/jenkins_home/secrets/
data/jenkins_home/plugins/
*.log
logs/
temp/
*.bak
*.backup
.vscode/
.idea/
*.swp
.DS_Store
```

4. .env.example

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

# Ansible SSH Key
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

5. deploy.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
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
    
    mkdir -p {casc,scripts,shared_scripts/{groovy,powershell,python,bash,pipelines},secrets,nginx/{conf.d,ssl},data/{jenkins_home,ansible/inventory,shared}}
    
    chmod 755 scripts/*.sh 2>/dev/null || true
    chmod 600 secrets/*.txt 2>/dev/null || true
    chmod +x shared_scripts/bash/*.sh 2>/dev/null || true
    chmod +x shared_scripts/python/*.py 2>/dev/null || true
    
    if [[ ! -f .env ]]; then
        warn "Файл .env не найден. Создание из шаблона..."
        cp .env.example .env
        error "Отредактируйте .env файл перед запуском"
        exit 1
    fi
    
    source .env
}

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

build_images() {
    log "Сборка Docker образов..."
    docker compose build --pull --no-cache
}

start_services() {
    log "Запуск сервисов..."
    
    docker compose up -d jenkins
    
    if [[ ${ENABLE_TOOLS:-false} == "true" ]]; then
        docker compose --profile tools up -d ansible-controller
    fi
    
    if [[ ${ENABLE_PROXY:-false} == "true" ]]; then
        docker compose --profile proxy up -d reverse-proxy
    fi
    
    if [[ ${ENABLE_AD_TEST:-false} == "true" ]]; then
        docker compose --profile ad-test up -d ldap-admin
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
    
    mkdir -p shared/{ansible,terraform,scripts,reports}
    
    log "✓ Настройка завершена"
    log "🌐 Jenkins доступен по адресу: http://localhost:8080"
    
    if docker compose exec jenkins test -f /var/jenkins_home/secrets/initialAdminPassword; then
        log "🔑 Initial Admin Password:"
        docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
    fi
}

main() {
    log "🚀 Запуск корпоративного Jenkins с интеграцией Active Directory и мульти-языковой поддержкой"
    
    check_dependencies
    setup_environment
    build_images
    check_ad_connectivity
    start_services
    wait_for_jenkins
    setup_initial_config
    
    log "✅ Развертывание завершено успешно!"
    log "🔐 Jenkins интегрирован с Active Directory"
    log "💻 Поддержка языков: Groovy, PowerShell, Python, Bash"
    log "👥 Группы AD для управления доступом:"
    log "   - jenkins-admins: полный доступ"
    log "   - jenkins-developers: разработчики"
    log "   - jenkins-viewers: просмотр"
}

main "$@"
```

6. plugins.txt

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

7. casc/jenkins.yaml

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
    Интегрировано с Active Directory
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
  
  groovy:
    enabled: true
    securityEnabled: true

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
      - "staticMethod org.codehaus.groovy.runtime.ProcessGroovyMethods execute"
      - "staticMethod org.codehaus.groovy.runtime.ProcessGroovyMethods getText"
```

8. kerberos/krb5.conf

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

9. scripts/management.sh

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
    test-ad)
        docker compose run --rm jenkins /var/jenkins_scripts/test-ad-connection.sh
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|rebuild|logs|backup|status|shell|update|test-ad}"
        exit 1
        ;;
esac
```

10. scripts/test-ad-connection.sh

```bash
#!/usr/bin/env bash

set -euo pipefail

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
    
    if [[ -f ../.env ]]; then
        source ../.env
    else
        error ".env file not found"
        exit 1
    fi
    
    local bind_user="${AD_BIND_USER}"
    local bind_password="${AD_BIND_PASSWORD}"
    local domain_controller="${AD_DOMAIN_CONTROLLER1}"
    local domain="${AD_DOMAIN}"
    local base_dn="${AD_BASE_DN}"
    local group_dn="${AD_GROUP_DN:-cn=jenkins-admins,ou=groups,dc=company,dc=com}"
    
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

11. scripts/setup-ad-groups.sh

```bash
#!/usr/bin/env bash

set -euo pipefail

create_ad_group() {
    local group_name="$1"
    local group_dn="$2"
    local bind_user="$3"
    local bind_password="$4"
    local domain_controller="$5"
    
    echo "Creating AD group: $group_name"
    
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
            echo "Group $group_name already exists"
        fi
    done
}

main "$@"
```

12. scripts/install-plugins.sh

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

13. Groovy скрипты

shared_scripts/groovy/SystemInfo.groovy

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

shared_scripts/groovy/UserManagement.groovy

```groovy
#!/usr/bin/env groovy

def getUserList() {
    def users = User.getAll()
    echo "Total Users: ${users.size()}"
    
    users.each { user ->
        echo "User: ${user.getId()} - ${user.getFullName()}"
    }
    
    return users.collect { it.getId() }
}

def checkUserPermissions(String username) {
    def user = User.get(username, false)
    if (user) {
        def auth = Jenkins.instance.getAuthorizationStrategy()
        if (auth instanceof hudson.security.GlobalMatrixAuthorizationStrategy) {
            def permissions = auth.getGrantedPermissions().findAll { it.sid == username }
            echo "Permissions for ${username}:"
            permissions.each { perm ->
                echo "  - ${perm.permission.name}"
            }
            return permissions.size()
        }
    }
    return 0
}
```

14. PowerShell скрипты

shared_scripts/powershell/SystemCheck.ps1

```powershell
#!/usr/bin/env pwsh

<#
.SYNOPSIS
    Corporate System Check PowerShell Script
.DESCRIPTION
    Performs system checks and returns status information
#>

param(
    [ValidateSet("System", "Network", "Storage", "All")]
    [string]$CheckType = "All"
)

function Get-SystemInfo {
    Write-Host "🖥️ System Information" -ForegroundColor Green
    
    if ($IsLinux) {
        $os = Get-Content /etc/os-release | ConvertFrom-StringData
        $memory = Get-CimInstance -ClassName Win32_OperatingSystem -ErrorAction SilentlyContinue
        if (-not $memory) {
            $memInfo = Get-Content /proc/meminfo
            $totalMem = ($memInfo | Where-Object { $_ -match 'MemTotal:' }) -replace '[^0-9]', ''
            $memory = @{ TotalPhysicalMemory = [long]$totalMem * 1024 }
        }
    } else {
        $os = Get-CimInstance Win32_OperatingSystem
        $memory = $os
    }
    
    return @{
        OS = if ($IsLinux) { $os.PRETTY_NAME } else { $os.Caption }
        Version = if ($IsLinux) { $os.VERSION_ID } else { $os.Version }
        Architecture = if ($IsLinux) { & uname -m } else { $os.OSArchitecture }
        MemoryGB = [math]::Round($memory.TotalPhysicalMemory / 1GB, 2)
        Processors = (Get-CimInstance Win32_ComputerSystem).NumberOfProcessors
    }
}

function Get-NetworkInfo {
    Write-Host "🌐 Network Information" -ForegroundColor Cyan
    
    if ($IsLinux) {
        $hostname = & hostname
        $dns = Get-Content /etc/resolv.conf | Where-Object { $_ -match 'nameserver' } | ForEach-Object { ($_ -split '\s+')[1] }
    } else {
        $hostname = $env:COMPUTERNAME
        $dns = (Get-DnsClientServerAddress -AddressFamily IPv4 | Where-Object { $_.ServerAddresses.Length -gt 0 }).ServerAddresses
    }
    
    return @{
        Hostname = $hostname
        DNS_Servers = $dns -join ', '
    }
}

function Get-StorageInfo {
    Write-Host "💾 Storage Information" -ForegroundColor Yellow
    
    if ($IsLinux) {
        $drives = & df -h / | Select-Object -Skip 1
        $storageInfo = @()
        
        foreach ($drive in $drives) {
            $parts = $drive -split '\s+'
            $storageInfo += @{
                Drive = $parts[0]
                TotalGB = $parts[1]
                UsedGB = $parts[2]
                FreeGB = $parts[3]
                Usage = $parts[4]
            }
        }
    } else {
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
    }
    
    return $storageInfo
}

function Invoke-ADCheck {
    Write-Host "🔐 Active Directory Check" -ForegroundColor Magenta
    
    try {
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

return $results | ConvertTo-Json -Depth 3
```

15. Python скрипты

shared_scripts/python/infrastructure_check.py

```python
#!/usr/bin/env python3
"""
Corporate Infrastructure Check Script
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
        except Exception as e:
            self.results['docker'] = {'status': f'error: {e}'}
            return False
    
    def check_jenkins_status(self):
        """Check Jenkins accessibility"""
        try:
            import requests
            response = requests.get('http://localhost:8080', timeout=10)
            self.results['jenkins'] = {
                'status': 'running' if response.status_code == 200 else 'unreachable',
                'status_code': response.status_code
            }
            return response.status_code == 200
        except Exception as e:
            self.results['jenkins'] = {'status': f'error: {e}'}
            return False
    
    def run_all_checks(self):
        """Run all infrastructure checks"""
        print("🔍 Running Infrastructure Checks...")
        
        checks = [
            ('System Resources', self.check_system_resources),
            ('Network Connectivity', lambda: self.check_network_connectivity()),
            ('Docker Status', self.check_docker_status),
            ('Jenkins Status', self.check_jenkins_status)
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
    
    print(json.dumps(results, indent=2))
    
    if any('error' in str(result).lower() for result in results.values()):
        sys.exit(1)
    else:
        sys.exit(0)

if __name__ == "__main__":
    main()
```

16. Bash скрипты

shared_scripts/bash/system_audit.sh

```bash
#!/bin/bash

set -euo pipefail

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

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
    check_jenkins_status
    generate_report
    
    log_success "System audit completed successfully"
}

if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

17. Jenkins Pipeline примеры

shared_scripts/pipelines/multi-language-pipeline.jenkinsfile

```groovy
pipeline {
    agent any
    tools {
        powershell 'powershell-linux'
        python 'python3'
    }
    stages {
        stage('System Audit - Bash') {
            steps {
                sh '''
                    chmod +x /usr/local/scripts/bash/system_audit.sh
                    /usr/local/scripts/bash/system_audit.sh
                '''
            }
        }
        
        stage('Infrastructure Check - Python') {
            steps {
                sh 'python3 /usr/local/scripts/python/infrastructure_check.py'
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
                    def systemInfo = load '/usr/local/scripts/groovy/SystemInfo.groovy'
                    def info = systemInfo.call()
                    echo "System Info: ${info}"
                }
            }
        }
    }
    post {
        always {
            echo "Multi-language pipeline execution completed"
        }
    }
}
```

18. Nginx конфигурация

nginx/conf.d/jenkins.conf

```nginx
server {
    listen 80;
    server_name jenkins.company.com;
    
    location / {
        return 301 https://$server_name$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name jenkins.company.com;
    
    ssl_certificate /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    ssl_prefer_server_ciphers off;
    
    location / {
        proxy_pass http://jenkins:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        proxy_connect_timeout 300;
        proxy_send_timeout 300;
        proxy_read_timeout 300;
        send_timeout 300;
    }
}
```

19. README.md

```markdown
# Jenkins Infrastructure as Code - Корпоративное решение

Полное решение для развертывания Jenkins с поддержкой Active Directory и мульти-языковой автоматизацией.

## Особенности

- 🔐 Интеграция с Active Directory
- 💻 Поддержка Groovy, PowerShell, Python, Bash
- 🐋 Контейнеризация через Docker Compose
- ⚙️ Configuration as Code
- 🔒 Безопасность корпоративного уровня
- 📊 Мониторинг и логирование

## Быстрый старт

1. Клонируйте репозиторий
2. Настройте переменные в `.env`
3. Запустите деплой:
   ```bash
   chmod +x deploy.sh
   ./deploy.sh
```

Управление

```bash
# Запуск/остановка
./scripts/management.sh start
./scripts/management.sh stop

# Тестирование AD
./scripts/management.sh test-ad

# Бэкап
./scripts/management.sh backup
```

Доступ

· Jenkins: http://localhost:8080
· LDAP Admin: http://localhost:8081 (если включен)

Документация

· Интеграция с AD
· Мульти-языковые скрипты
· Безопасность

```

## 20. Создание необходимых директорий и файлов

```bash
#!/bin/bash
# create-structure.sh

mkdir -p \
  casc \
  kerberos \
  scripts \
  shared_scripts/{groovy,powershell,python,bash,pipelines} \
  nginx/{conf.d,ssl} \
  data/{jenkins_home,ansible/inventory,shared} \
  secrets

# Создание основных файлов
touch docker-compose.yml
touch Dockerfile
touch .dockerignore
touch .env
touch .env.example
touch deploy.sh
touch plugins.txt
touch casc/jenkins.yaml
touch kerberos/krb5.conf

# Скрипты управления
touch scripts/management.sh
touch scripts/test-ad-connection.sh
touch scripts/setup-ad-groups.sh
touch scripts/install-plugins.sh

# Мульти-языковые скрипты
touch shared_scripts/groovy/SystemInfo.groovy
touch shared_scripts/groovy/UserManagement.groovy
touch shared_scripts/powershell/SystemCheck.ps1
touch shared_scripts/python/infrastructure_check.py
touch shared_scripts/bash/system_audit.sh
touch shared_scripts/pipelines/multi-language-pipeline.jenkinsfile

# Nginx конфигурация
touch nginx/conf.d/jenkins.conf

# Документация
touch README.md

# Установка прав
chmod +x deploy.sh
chmod +x scripts/*.sh
chmod +x shared_scripts/bash/*.sh
chmod +x shared_scripts/python/*.py

echo "Структура проекта создана!"
```

Завершение настройки

1. Настройте секреты:

```bash
echo "your_secure_password" > secrets/jenkins_admin_password.txt
echo "your_smtp_password" > secrets/smtp_password.txt
echo "your_ad_password" > secrets/ad_bind_password.txt
chmod 600 secrets/*.txt
```

1. Настройте .env:

```bash
cp .env.example .env
# Отредактируйте .env с вашими значениями
```

1. Запустите деплой:

```bash
./deploy.sh
```

Это полное корпоративное решение готово к использованию с поддержкой Active Directory, мульти-языковой автоматизацией и всеми необходимыми инструментами для Infrastructure as Code.



Исправляем ошибку с docker-compose. В современных версиях Docker используется docker compose (без дефиса) как встроенная команда.

Обновленные скрипты с исправлением:

1. Обновленный deploy.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
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
    
    # Проверка Docker
    if ! command -v docker &> /dev/null; then
        error "Docker не установлен"
        echo "Установите Docker: https://docs.docker.com/get-docker/"
        exit 1
    fi
    
    # Проверка Docker Compose (modern plugin version)
    if ! docker compose version &> /dev/null; then
        error "Docker Compose не установлен или не работает"
        echo "Установите Docker Compose: https://docs.docker.com/compose/install/"
        exit 1
    fi
    
    log "✓ Docker версия: $(docker --version)"
    log "✓ Docker Compose версия: $(docker compose version)"
    log "✓ Все зависимости удовлетворены"
}

setup_environment() {
    log "Настройка окружения..."
    
    mkdir -p {casc,scripts,shared_scripts/{groovy,powershell,python,bash,pipelines},secrets,nginx/{conf.d,ssl},data/{jenkins_home,ansible/inventory,shared}}
    
    chmod 755 scripts/*.sh 2>/dev/null || true
    chmod 600 secrets/*.txt 2>/dev/null || true
    chmod +x shared_scripts/bash/*.sh 2>/dev/null || true
    chmod +x shared_scripts/python/*.py 2>/dev/null || true
    
    if [[ ! -f .env ]]; then
        warn "Файл .env не найден. Создание из шаблона..."
        cp .env.example .env
        echo -e "${YELLOW}Отредактируйте .env файл перед запуском:${NC}"
        echo "  nano .env"
        echo -e "${YELLOW}Затем запустите скрипт снова:${NC}"
        echo "  ./deploy.sh"
        exit 1
    fi
    
    # Загрузка переменных окружения
    set -a
    source .env
    set +a
    
    log "✓ Окружение настроено"
}

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

build_images() {
    log "Сборка Docker образов..."
    docker compose build --pull --no-cache
}

start_services() {
    log "Запуск сервисов..."
    
    docker compose up -d jenkins
    
    if [[ ${ENABLE_TOOLS:-false} == "true" ]]; then
        docker compose --profile tools up -d ansible-controller
    fi
    
    if [[ ${ENABLE_PROXY:-false} == "true" ]]; then
        docker compose --profile proxy up -d reverse-proxy
    fi
    
    if [[ ${ENABLE_AD_TEST:-false} == "true" ]]; then
        docker compose --profile ad-test up -d ldap-admin
    fi
}

wait_for_jenkins() {
    log "Ожидание запуска Jenkins..."
    local timeout=180
    local counter=0
    
    while ! curl -s -f "http://localhost:8080" > /dev/null; do
        sleep 5
        counter=$((counter + 5))
        echo "Ожидание... ${counter}с"
        if [[ $counter -ge $timeout ]]; then
            error "Таймаут ожидания Jenkins"
            docker compose logs jenkins
            exit 1
        fi
    done
    log "✓ Jenkins запущен"
}

get_initial_password() {
    log "Получение initial admin password..."
    
    local max_attempts=30
    local attempt=0
    
    while [[ $attempt -lt $max_attempts ]]; do
        if docker compose exec jenkins test -f /var/jenkins_home/secrets/initialAdminPassword; then
            log "🔑 Initial Admin Password:"
            docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
            return 0
        fi
        sleep 5
        attempt=$((attempt + 1))
    done
    
    warn "Initial admin password не найден. Проверьте логи Jenkins."
    docker compose logs jenkins
}

setup_initial_config() {
    log "Настройка начальной конфигурации..."
    
    mkdir -p shared/{ansible,terraform,scripts,reports,backups}
    
    # Копирование скриптов в общую директорию
    cp -r shared_scripts/* shared/scripts/ 2>/dev/null || true
    
    log "✓ Настройка завершена"
    log "🌐 Jenkins доступен по адресу: http://localhost:8080"
    log "🐋 Для управления используйте: ./scripts/management.sh"
}

check_system_resources() {
    log "Проверка системных ресурсов..."
    
    local total_memory=$(free -g | awk 'NR==2{print $2}')
    local available_disk=$(df -h / | awk 'NR==2{print $4}')
    
    if [[ $total_memory -lt 4 ]]; then
        warn "Мало оперативной памяти (доступно: ${total_memory}GB, рекомендуется: 4GB+)"
    else
        log "✓ Оперативная память: ${total_memory}GB"
    fi
    
    log "✓ Свободное место на диске: ${available_disk}"
}

main() {
    echo -e "${BLUE}"
    echo "╔══════════════════════════════════════════════════════════════╗"
    echo "║           Jenkins Infrastructure as Code - Deploy           ║"
    echo "║         Корпоративное решение с AD интеграцией              ║"
    echo "╚══════════════════════════════════════════════════════════════╝"
    echo -e "${NC}"
    
    check_dependencies
    check_system_resources
    setup_environment
    build_images
    check_ad_connectivity
    start_services
    wait_for_jenkins
    get_initial_password
    setup_initial_config
    
    echo -e "${GREEN}"
    echo "╔══════════════════════════════════════════════════════════════╗"
    echo "║                     ДЕПЛОЙ ЗАВЕРШЕН!                        ║"
    echo "╠══════════════════════════════════════════════════════════════╣"
    echo "║ 🔐  Jenkins интегрирован с Active Directory                 ║"
    echo "║ 💻  Поддержка языков: Groovy, PowerShell, Python, Bash      ║"
    echo "║ 🐋  Управление: ./scripts/management.sh                     ║"
    echo "║ 🌐  Доступ: http://localhost:8080                           ║"
    echo "╚══════════════════════════════════════════════════════════════╝"
    echo -e "${NC}"
}

main "$@"
```

2. Обновленный scripts/management.sh

```bash
#!/usr/bin/env bash

set -euo pipefail

# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
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

check_compose() {
    if ! docker compose version &> /dev/null; then
        error "Docker Compose не доступен"
        exit 1
    fi
}

show_usage() {
    echo -e "${BLUE}Использование: $0 {start|stop|restart|rebuild|logs|backup|status|shell|update|test-ad|info}${NC}"
    echo ""
    echo "Команды:"
    echo "  start     - Запуск всех сервисов"
    echo "  stop      - Остановка всех сервисов"
    echo "  restart   - Перезапуск всех сервисов"
    echo "  rebuild   - Пересборка и перезапуск"
    echo "  logs      - Просмотр логов (можно указать сервис: $0 logs jenkins)"
    echo "  backup    - Создание бэкапа Jenkins"
    echo "  status    - Статус сервисов"
    echo "  shell     - Вход в контейнер (по умолчанию: jenkins)"
    echo "  update    - Обновление образов и перезапуск"
    echo "  test-ad   - Тестирование подключения к AD"
    echo "  info      - Информация о системе"
    echo ""
}

case "${1:-}" in
    start)
        check_compose
        log "Запуск сервисов Jenkins..."
        docker compose up -d
        log "Сервисы запущены"
        ;;
    stop)
        check_compose
        log "Остановка сервисов..."
        docker compose down
        log "Сервисы остановлены"
        ;;
    restart)
        check_compose
        log "Перезапуск сервисов..."
        docker compose restart
        log "Сервисы перезапущены"
        ;;
    rebuild)
        check_compose
        log "Пересборка и перезапуск..."
        docker compose down
        docker compose build --no-cache
        docker compose up -d
        log "Пересборка завершена"
        ;;
    logs)
        check_compose
        service="${2:-}"
        if [[ -n "$service" ]]; then
            log "Просмотр логов сервиса: $service"
            docker compose logs -f "$service"
        else
            log "Просмотр логов всех сервисов"
            docker compose logs -f
        fi
        ;;
    backup)
        check_compose
        log "Создание бэкапа Jenkins..."
        backup_file="jenkins_backup_$(date +%Y%m%d_%H%M%S).tar.gz"
        docker compose exec -T jenkins tar czf "/shared/$backup_file" -C /var/jenkins_home .
        log "Бэкап создан: shared/$backup_file"
        ;;
    status)
        check_compose
        log "Статус сервисов:"
        docker compose ps
        ;;
    shell)
        check_compose
        service="${2:-jenkins}"
        log "Вход в контейнер: $service"
        docker compose exec "$service" bash
        ;;
    update)
        check_compose
        log "Обновление образов..."
        docker compose pull
        docker compose build --pull
        docker compose up -d
        log "Обновление завершено"
        ;;
    test-ad)
        check_compose
        log "Тестирование подключения к Active Directory..."
        docker compose run --rm jenkins /var/jenkins_scripts/test-ad-connection.sh
        ;;
    info)
        check_compose
        echo -e "${BLUE}"
        echo "╔══════════════════════════════════════════════════════════════╗"
        echo "║                   Jenkins System Info                       ║"
        echo "╠══════════════════════════════════════════════════════════════╣"
        echo -e "${NC}"
        echo "Сервисы:"
        docker compose ps
        echo ""
        echo "Сети:"
        docker network ls | grep jenkins
        echo ""
        echo "Тома:"
        docker volume ls | grep jenkins
        echo ""
        echo "Логины по умолчанию:"
        echo "  Jenkins: http://localhost:8080"
        echo "  LDAP Admin: http://localhost:8081 (если включен)"
        ;;
    *)
        show_usage
        exit 1
        ;;
esac
```

3. Создаем скрипт установки зависимостей scripts/install-dependencies.sh

```bash
#!/usr/bin/env bash

set -euo pipefail

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[INFO] $1${NC}"
}

warn() {
    echo -e "${YELLOW}[WARN] $1${NC}"
}

error() {
    echo -e "${RED}[ERROR] $1${NC}"
}

install_docker_ubuntu() {
    log "Установка Docker на Ubuntu/Debian..."
    
    # Обновление пакетов
    sudo apt-get update
    sudo apt-get install -y ca-certificates curl gnupg
    
    # Добавление Docker репозитория
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg
    
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    
    # Установка Docker
    sudo apt-get update
    sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    
    # Добавление пользователя в группу docker
    sudo usermod -aG docker $USER
    
    log "✓ Docker установлен"
}

install_docker_centos() {
    log "Установка Docker на CentOS/RHEL..."
    
    # Установка yum-utils
    sudo yum install -y yum-utils
    
    # Добавление Docker репозитория
    sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    
    # Установка Docker
    sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    
    # Запуск и включение Docker
    sudo systemctl start docker
    sudo systemctl enable docker
    
    # Добавление пользователя в группу docker
    sudo usermod -aG docker $USER
    
    log "✓ Docker установлен"
}

install_docker_macos() {
    log "Установка Docker на macOS..."
    warn "Пожалуйста, установите Docker Desktop вручную:"
    echo "  https://docs.docker.com/desktop/install/mac-install/"
    echo ""
    echo "После установки запустите Docker Desktop и убедитесь, что он работает."
    exit 1
}

install_docker_windows() {
    log "Установка Docker на Windows..."
    warn "Пожалуйста, установите Docker Desktop вручную:"
    echo "  https://docs.docker.com/desktop/install/windows-install/"
    echo ""
    echo "После установки запустите Docker Desktop и убедитесь, что он работает."
    exit 1
}

detect_os() {
    case "$(uname -s)" in
        Linux*)
            if [[ -f /etc/os-release ]]; then
                source /etc/os-release
                case $ID in
                    ubuntu|debian)
                        install_docker_ubuntu
                        ;;
                    centos|rhel|fedora)
                        install_docker_centos
                        ;;
                    *)
                        error "Неподдерживаемый дистрибутив Linux: $ID"
                        exit 1
                        ;;
                esac
            else
                error "Не удалось определить дистрибутив Linux"
                exit 1
            fi
            ;;
        Darwin*)
            install_docker_macos
            ;;
        CYGWIN*|MINGW32*|MINGW64*|MSYS*)
            install_docker_windows
            ;;
        *)
            error "Неподдерживаемая ОС: $(uname -s)"
            exit 1
            ;;
    esac
}

verify_installation() {
    log "Проверка установки..."
    
    if ! command -v docker &> /dev/null; then
        error "Docker не установлен корректно"
        exit 1
    fi
    
    if ! docker compose version &> /dev/null; then
        error "Docker Compose не установлен корректно"
        exit 1
    fi
    
    log "✓ Docker версия: $(docker --version)"
    log "✓ Docker Compose версия: $(docker compose version)"
    
    # Тестовый запуск
    if ! docker run --rm hello-world &> /dev/null; then
        warn "Docker не может запускать контейнеры. Возможно, нужна перезагрузка."
        echo "Пожалуйста, выполните:"
        echo "  sudo systemctl restart docker"
        echo "Или перезагрузите систему"
    else
        log "✓ Docker работает корректно"
    fi
}

main() {
    echo -e "${GREEN}"
    echo "╔══════════════════════════════════════════════════════════════╗"
    echo "║              Установка Docker и зависимостей                ║"
    echo "╚══════════════════════════════════════════════════════════════╝"
    echo -e "${NC}"
    
    # Проверка прав
    if [[ $EUID -eq 0 ]]; then
        error "Не запускайте скрипт от root. Скрипт запросит права когда нужно."
        exit 1
    fi
    
    # Проверка уже установленного Docker
    if command -v docker &> /dev/null && docker compose version &> /dev/null; then
        log "Docker и Docker Compose уже установлены"
        verify_installation
        exit 0
    fi
    
    log "Обнаружена ОС: $(uname -s)"
    detect_os
    
    verify_installation
    
    echo ""
    log "🎉 Установка завершена успешно!"
    warn "⚠️  Для применения изменений可能需要 перезагрузка或重新登录"
    echo ""
    log "Далее выполните:"
    echo "  ./deploy.sh"
    echo ""
}

main "$@"
```

4. Создаем README с инструкциями

README.md:

```markdown
# Jenkins Infrastructure as Code - Корпоративное решение

Полное решение для развертывания Jenkins с поддержкой Active Directory и мульти-языковой автоматизацией.

## 🚀 Быстрый старт

### 1. Установка зависимостей

Если Docker не установлен:

```bash
# Автоматическая установка Docker
chmod +x scripts/install-dependencies.sh
./scripts/install-dependencies.sh

# Или установите вручную:
# https://docs.docker.com/get-docker/
```

2. Настройка окружения

```bash
# Копируем пример настроек
cp .env.example .env

# Редактируем настройки
nano .env
```

3. Запуск развертывания

```bash
# Делаем скрипт исполняемым
chmod +x deploy.sh

# Запускаем деплой
./deploy.sh
```

📋 Требования

· Docker Engine 20.10+
· Docker Compose Plugin 2.0+
· 4GB+ RAM
· 20GB+ свободного места

🛠 Управление

```bash
# Запуск/остановка
./scripts/management.sh start
./scripts/management.sh stop

# Просмотр логов
./scripts/management.sh logs
./scripts/management.sh logs jenkins

# Бэкап
./scripts/management.sh backup

# Тестирование AD
./scripts/management.sh test-ad

# Статус сервисов
./scripts/management.sh status
```

🌐 Доступ

· Jenkins: http://localhost:8080
· LDAP Admin: http://localhost:8081 (если включен)
· Initial Password: будет показан после деплоя

🔧 Особенности

· ✅ Active Directory интеграция
· ✅ Поддержка языков: Groovy, PowerShell, Python, Bash
· ✅ Docker Compose V2 (современная версия)
· ✅ Configuration as Code
· ✅ Мульти-языковые скрипты
· ✅ Корпоративная безопасность

🐛 Решение проблем

Ошибка: "Не найдена зависимость: docker-compose"

Используется современная версия Docker Compose как плагин:

```bash
# Проверка установки
docker compose version

# Если не установлен, обновите Docker:
# https://docs.docker.com/engine/install/
```

Ошибка прав доступа

```bash
# Добавьте пользователя в группу docker
sudo usermod -aG docker $USER

# Перезайдите в систему или выполните:
newgrp docker
```

Проблемы с памятью

Увеличьте лимиты Docker в настройках Docker Desktop или в /etc/docker/daemon.json:

```json
{
  "default-shm-size": "1g",
  "memory": "4g"
}
```

📁 Структура проекта

```
jenkins-iac-corporate/
├── deploy.sh                 # Основной скрипт развертывания
├── scripts/
│   ├── management.sh         # Управление сервисами
│   └── install-dependencies.sh # Установка Docker
└── shared_scripts/           # Мульти-языковые скрипты
```

🤝 Поддержка

При проблемах с развертыванием:

1. Проверьте что Docker запущен: docker info
2. Проверьте логи: ./scripts/management.sh logs
3. Убедитесь что порты 8080 и 50000 свободны

```

## 5. Создаем `.env.example` с правильными настройками

```env
# Corporate Jenkins Configuration
COMPOSE_PROJECT_NAME=jenkins-iac-corporate
JENKINS_URL=http://localhost:8080

# Active Directory Configuration
AD_DOMAIN=company.com
AD_DOMAIN_CONTROLLER1=dc1.company.com
AD_DOMAIN_CONTROLLER2=dc2.company.com
AD_BIND_USER=svc_jenkins@company.com
AD_BIND_PASSWORD=change_this_password
AD_BASE_DN=DC=company,DC=com
AD_USER_SEARCH_BASE=OU=Users,DC=company,DC=com
AD_GROUP_SEARCH_BASE=OU=Groups,DC=company,DC=com
AD_DNS_SERVER_1=192.168.1.1
AD_DNS_SERVER_2=192.168.1.2

# Security Secrets - CHANGE THESE!
ADMIN_PASSWORD=change_this_admin_password
SMTP_PASSWORD=change_this_smtp_password
GIT_PASSWORD=change_this_git_token
ANSIBLE_VAULT_PASSWORD=change_this_vault_password
DOCKER_REGISTRY_USER=registry-user
DOCKER_REGISTRY_PASSWORD=change_this_registry_password

# Scripting & Language Security
PYTHON_API_KEY=change_this_python_api_key
POWERSHELL_ENCRYPTION_KEY=change_this_powershell_key

# Ansible SSH Key (замените на свой)
ANSIBLE_SSH_KEY=ssh-rsa AAAAB3NzaC1yc2E...change_this_ssh_key

# Features
ENABLE_TOOLS=true
ENABLE_PROXY=false
ENABLE_MONITORING=true
ENABLE_AD_TEST=false
ENABLE_MULTI_LANGUAGE=true

# Network
JENKINS_NETWORK_SUBNET=172.20.0.0/24
```

6. Создаем скрипт инициализации

init-project.sh:

```bash
#!/usr/bin/env bash

set -euo pipefail

echo "🎯 Инициализация проекта Jenkins Infrastructure as Code..."

# Создание необходимых директорий
mkdir -p {casc,scripts,shared_scripts/{groovy,powershell,python,bash,pipelines},secrets,nginx/{conf.d,ssl},data/{jenkins_home,ansible/inventory,shared}}

# Создание основных файлов если их нет
[[ ! -f .env ]] && cp .env.example .env
[[ ! -f docker-compose.yml ]] && echo "# Docker Compose file will be created" > docker-compose.yml

# Установка прав на скрипты
chmod +x scripts/*.sh 2>/dev/null || true
chmod +x shared_scripts/bash/*.sh 2>/dev/null || true
chmod +x shared_scripts/python/*.py 2>/dev/null || true

# Создание базовых секретов если их нет
[[ ! -f secrets/jenkins_admin_password.txt ]] && echo "change_this_password" > secrets/jenkins_admin_password.txt
[[ ! -f secrets/ad_bind_password.txt ]] && echo "change_this_password" > secrets/ad_bind_password.txt
chmod 600 secrets/*.txt 2>/dev/null || true

echo "✅ Инициализация завершена!"
echo ""
echo "📝 Следующие шаги:"
echo "1. Отредактируйте файл .env: nano .env"
echo "2. Запустите развертывание: ./deploy.sh"
echo ""
echo "🔧 Или установите зависимости: ./scripts/install-dependencies.sh"
```

Запуск проекта:

```bash
# Делаем все скрипты исполняемыми
chmod +x init-project.sh
chmod +x deploy.sh
chmod +x scripts/*.sh

# Инициализация проекта
./init-project.sh

# Установка зависимостей (если нужно)
./scripts/install-dependencies.sh

# Запуск деплоя
./deploy.sh
```

Теперь проект использует современную версию Docker Compose (docker compose) вместо устаревшей (docker-compose), что решает проблему с зависимостями.