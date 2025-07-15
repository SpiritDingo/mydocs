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