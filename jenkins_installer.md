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