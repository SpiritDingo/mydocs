Вот обновлённый `docker-compose.yml` с добавлением Nginx в качестве reverse proxy для Nexus:

```yaml
version: '3.8'

services:
  nexus:
    image: sonatype/nexus3:3.80.0
    container_name: nexus
    restart: unless-stopped
    user: "200:200"
    volumes:
      - /opt/nexus-data:/nexus-data
    expose:
      - "8081"
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms2g -Xmx2g -XX:MaxDirectMemorySize=2g
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081"]
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - nexus-network

  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/certs:/etc/nginx/certs
    depends_on:
      - nexus
    networks:
      - nexus-network

networks:
  nexus-network:
    driver: bridge
```

Вам также нужно создать директорию `nginx/conf.d` и добавить конфигурационный файл для Nexus. Пример конфигурации Nginx (`nginx/conf.d/nexus.conf`):

```nginx
upstream nexus {
    server nexus:8081;
}

server {
    listen 80;
    server_name your-domain.com; # Замените на ваш домен или IP

    location / {
        proxy_pass http://nexus;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Раскомментируйте для HTTPS (нужен сертификат)
# server {
#     listen 443 ssl;
#     server_name your-domain.com;
#
#     ssl_certificate /etc/nginx/certs/your-cert.pem;
#     ssl_certificate_key /etc/nginx/certs/your-key.key;
#
#     location / {
#         proxy_pass http://nexus;
#         proxy_set_header Host $host;
#         proxy_set_header X-Real-IP $remote_addr;
#         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#         proxy_set_header X-Forwarded-Proto $scheme;
#     }
# }
```

Основные изменения:
1. Заменил `ports` на `expose` для Nexus, так как теперь доступ будет через Nginx
2. Добавил сеть `nexus-network` для связи между контейнерами
3. Добавил сервис Nginx с пробросом портов 80 и 443
4. Добавил volumes для конфигурации Nginx и сертификатов (если будете использовать HTTPS)

После настройки:
1. Создайте директории: `mkdir -p nginx/{conf.d,certs}`
2. Поместите конфигурацию Nginx в `nginx/conf.d/nexus.conf`
3. Запустите: `docker-compose up -d`