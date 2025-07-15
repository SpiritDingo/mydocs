Вот пример `docker-compose.yml` для Jenkins с предустановленными плагинами, сохранением конфигурации и плагинов через volumes:

```yaml
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    privileged: true
    user: root
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
    networks:
      - jenkins_network

volumes:
  jenkins_home:

networks:
  jenkins_network:
    driver: bridge
```

Для предустановки плагинов создайте файл `Dockerfile`:

```dockerfile
FROM jenkins/jenkins:lts

# Отключаем setup wizard
ENV JAVA_OPTS -Djenkins.install.runSetupWizard=false

# Устанавливаем плагины
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli -f /usr/share/jenkins/ref/plugins.txt
```

И файл `plugins.txt` со списком плагинов (пример):

```
git
github
blueocean
docker-workflow
pipeline-aws
kubernetes
credentials-binding
ssh-slaves
ws-cleanup
timestamper
ansible
```

Соберите образ с плагинами:

```bash
docker build -t my-jenkins .
```

Затем обновите `docker-compose.yml` для использования вашего образа:

```yaml
services:
  jenkins:
    image: my-jenkins
    # остальное без изменений
```

После запуска (`docker-compose up -d`) Jenkins будет:
- Доступен на порту 8080
- С предустановленными плагинами
- Сохранять все данные в volume `jenkins_home`
- Иметь доступ к Docker соксу для работы с Docker

Для первоначальной настройки пароля админа проверьте логи контейнера:
```bash
docker logs jenkins
```


Вот расширенный **production-ready** вариант `docker-compose.yml` для Jenkins в компании, с:  
- **Предустановленными плагинами** (включая популярные для CI/CD)  
- **Volume для конфигурации, плагинов и сборок**  
- **Настройкой памяти и CPU**  
- **Докер-интеграцией** (Docker-in-Docker)  
- **Резервным копированием**  
- **Nginx в качестве reverse proxy с HTTPS**  
- **Поддержкой агентов (Jenkins agents)**  

---

### 1. **`docker-compose.yml`**  
```yaml
version: '3.8'

services:
  jenkins:
    image: custom-jenkins  # Будет собран из Dockerfile
    container_name: jenkins
    restart: unless-stopped
    user: root
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false -Xmx4g -Xms2g
      - JENKINS_ADMIN_ID=admin
      - JENKINS_ADMIN_PASSWORD=securepassword123  # Замените на свой
    volumes:
      - jenkins_data:/var/jenkins_home
      - docker_sock:/var/run/docker.sock
    ports:
      - "50000:50000"  # Для агентов
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
    networks:
      - jenkins_network

  # Reverse Proxy (Nginx + HTTPS)
  nginx:
    image: nginx:alpine
    container_name: jenkins_nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - jenkins
    networks:
      - jenkins_network

  # Jenkins Agent (для распределенных сборок)
  jenkins-agent:
    image: jenkins/ssh-agent:latest
    container_name: jenkins_agent
    environment:
      - JENKINS_AGENT_SSH_PUBKEY=ssh-rsa AAAAB3Nza...  # Ваш публичный ключ
    networks:
      - jenkins_network

volumes:
  jenkins_data:
  docker_sock:

networks:
  jenkins_network:
    driver: bridge
```

---

### 2. **`Dockerfile`** (с предустановленными плагинами)  
```dockerfile
FROM jenkins/jenkins:lts-jdk17

# Отключаем setup wizard
ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"

# Устанавливаем плагины (список ниже)
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli -f /usr/share/jenkins/ref/plugins.txt

# Настройка Groovy-скриптов для автоматической конфигурации
COPY init.groovy.d/ /usr/share/jenkins/ref/init.groovy.d/
```

---

### 3. **`plugins.txt`** (список плагинов)  
```
git
github
blueocean
docker-plugin
docker-workflow
kubernetes
pipeline-aws
credentials-binding
ssh-slaves
ws-cleanup
timestamper
ansible
sonarqube
terraform
nodejs
email-ext
metrics
```

---

### 4. **`nginx/conf.d/jenkins.conf`** (HTTPS Proxy)  
```nginx
upstream jenkins {
  server jenkins:8080;
}

server {
  listen 80;
  server_name jenkins.your-company.com;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  server_name jenkins.your-company.com;

  ssl_certificate /etc/nginx/ssl/cert.pem;
  ssl_certificate_key /etc/nginx/ssl/key.pem;

  location / {
    proxy_pass http://jenkins;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

---

### 5. **`init.groovy.d/security.groovy`** (базовая безопасность)  
```groovy
import jenkins.model.*
import hudson.security.*

def instance = Jenkins.getInstance()

// Включаем безопасность
def hudsonRealm = new HudsonPrivateSecurityRealm(false)
hudsonRealm.createAccount(System.getenv("JENKINS_ADMIN_ID"), System.getenv("JENKINS_ADMIN_PASSWORD"))
instance.setSecurityRealm(hudsonRealm)

// Включаем авторизацию
def strategy = new GlobalMatrixAuthorizationStrategy()
strategy.add(Jenkins.ADMINISTER, System.getenv("JENKINS_ADMIN_ID"))
instance.setAuthorizationStrategy(strategy)

instance.save()
```

---

### **Запуск**  
1. Создайте SSL-сертификаты (например, через Let's Encrypt):  
   ```bash
   mkdir -p nginx/ssl && openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout nginx/ssl/key.pem -out nginx/ssl/cert.pem
   ```
2. Соберите образ Jenkins:  
   ```bash
   docker build -t custom-jenkins .
   ```
3. Запустите:  
   ```bash
   docker-compose up -d
   ```

---

### **Дополнительные улучшения**  
✅ **Резервное копирование** (через `rsync` или `borgbackup` для `jenkins_data`)  
✅ **Мониторинг** (Prometheus + Grafana)  
✅ **Кластеризация** (Jenkins + Kubernetes)  
✅ **Интеграция с Vault** (для секретов)  

Этот вариант подходит для **корпоративного использования** с балансом между безопасностью и удобством. Настройки можно кастомизировать под нужды компании.



Вот **расширенный корпоративный вариант** Jenkins в Docker, включающий **резервное копирование, мониторинг, кластеризацию через Kubernetes, интеграцию с HashiCorp Vault** и другие production-фичи.

---

## 1. Обновлённый `docker-compose.yml` (с мониторингом и резервным копированием)
```yaml
version: '3.8'

services:
  jenkins:
    image: custom-jenkins  # Собранный образ с плагинами
    container_name: jenkins
    restart: unless-stopped
    user: root
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false -Xmx4g -Xms2g
      - VAULT_ADDR=http://vault:8200  # Интеграция с Vault
    volumes:
      - jenkins_data:/var/jenkins_home
      - docker_sock:/var/run/docker.sock
    ports:
      - "50000:50000"
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
    networks:
      - jenkins_network

  # Reverse Proxy (Nginx + HTTPS)
  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/ssl:/etc/nginx/ssl
    ports:
      - "80:80"
      - "443:443"
    networks:
      - jenkins_network

  # HashiCorp Vault для управления секретами
  vault:
    image: vault:latest
    environment:
      - VAULT_DEV_ROOT_TOKEN_ID=myroot
      - VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200
    ports:
      - "8200:8200"
    volumes:
      - vault_data:/vault/file
    networks:
      - jenkins_network

  # Prometheus + Grafana для мониторинга
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    networks:
      - jenkins_network

  grafana:
    image: grafana/grafana
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - jenkins_network

  # Jenkins Agent (Kubernetes-based)
  jenkins-agent:
    image: jenkins/inbound-agent:latest
    environment:
      - KUBERNETES_NAMESPACE=jenkins
    networks:
      - jenkins_network

volumes:
  jenkins_data:
  docker_sock:
  vault_data:
  grafana_data:

networks:
  jenkins_network:
    driver: bridge
```

---

## 2. **Резервное копирование (BorgBackup + Cron)**
Добавьте в `Dockerfile`:
```dockerfile
RUN apt-get update && apt-get install -y borgbackup
```

Пример скрипта `backup.sh`:
```bash
#!/bin/bash
export BORG_PASSPHRASE="yourpassphrase"
borg create --stats /mnt/backups/jenkins::'{now:%Y-%m-%d}' /var/jenkins_home
borg prune --keep-daily 7 --keep-weekly 4 /mnt/backups/jenkins
```

Добавьте в `docker-compose.yml`:
```yaml
  backup:
    image: alpine
    volumes:
      - jenkins_data:/source
      - backup_volume:/backup
    command: sh -c "echo '0 2 * * * /backup/backup.sh' | crontab - && crond -f"
```

---

## 3. **Мониторинг (Prometheus + Grafana)**
`prometheus.yml`:
```yaml
scrape_configs:
  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['jenkins:8080']
```

Настройки Grafana:
- Импортируйте дашборд **Jenkins Performance and Health Overview** (ID: **9964**)
- Добавьте источник данных Prometheus (`http://prometheus:9090`)

---

## 4. **Кластеризация (Jenkins + Kubernetes)**
1. Установите плагин **Kubernetes** в `plugins.txt`:
   ```
   kubernetes
   ```

2. Настройте `init.groovy.d/k8s.groovy`:
   ```groovy
   import org.csanchez.jenkins.plugins.kubernetes.*
   import jenkins.model.*

   def instance = Jenkins.getInstance()
   def k8sCloud = new KubernetesCloud(
     "jenkins-k8s",
     null,
     "https://kubernetes.default.svc",
     "jenkins-agent",
     "default"
   )
   instance.clouds.add(k8sCloud)
   instance.save()
   ```

---

## 5. **Интеграция с HashiCorp Vault**
1. Установите плагин **HashiCorp Vault** в `plugins.txt`:
   ```
   hashicorp-vault-plugin
   ```

2. Настройте `init.groovy.d/vault.groovy`:
   ```groovy
   import com.datapipe.jenkins.vault.configuration.*
   import jenkins.model.*

   def instance = Jenkins.getInstance()
   def vaultConfig = new VaultConfiguration(
     "http://vault:8200",
     "myroot"
   )
   instance.getExtensionList(VaultConfiguration.class).add(vaultConfig)
   instance.save()
   ```

---

## 6. **Запуск и проверка**
```bash
# Собрать и запустить
docker-compose build
docker-compose up -d

# Проверить логи
docker-compose logs -f jenkins
```

---

### **Итоговая архитектура**
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Jenkins   │───▶│ Kubernetes  │───▶│  Docker     │
└─────────────┘    └─────────────┘    └─────────────┘
       │                  │                  │
       ▼                  ▼                  ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Vault     │    │ Prometheus  │    │  Nginx      │
└─────────────┘    └─────────────┘    └─────────────┘
       │                  │
       ▼                  ▼
┌─────────────┐    ┌─────────────┐
│   BorgBackup│    │  Grafana    │
└─────────────┘    └─────────────┘
```

Это решение покрывает все ключевые аспекты enterprise-развёртывания Jenkins:
- **Отказоустойчивость** (Kubernetes + резервные копии)
- **Безопасность** (Vault + HTTPS)
- **Мониторинг** (Prometheus/Grafana)
- **Масштабируемость** (Kubernetes agents)