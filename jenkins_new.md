
Похоже, у вас возникли проблемы с развертыванием Jenkins через Docker Compose. Давайте исправим эти ошибки и предоставим полное рабочее решение.

## Исправленный docker-compose.yml

Устраним проблемы с версией и переменными:

```yaml
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: corp-jenkins
    user: root
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_data:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/local/bin/docker:/usr/local/bin/docker
    environment:
      - JENKINS_ADMIN_ID=admin
      - JENKINS_ADMIN_PASSWORD=password
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
    restart: unless-stopped
    networks:
      - jenkins-net

  agent:
    image: jenkins/inbound-agent
    container_name: jenkins-agent
    environment:
      - JENKINS_URL=http://jenkins:8080
      - JENKINS_AGENT_NAME=agent1
      - JENKINS_SECRET=${AGENT_SECRET:-default-secret}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - jenkins
    networks:
      - jenkins-net

networks:
  jenkins-net:
    driver: bridge

volumes:
  jenkins_data:
```

## Dockerfile для кастомного образа Jenkins

Создайте файл `docker/jenkins/Dockerfile`:

```dockerfile
FROM jenkins/jenkins:lts

USER root

# Установка необходимых пакетов
RUN apt-get update && apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    software-properties-common

# Установка Docker
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add - && \
    add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable" && \
    apt-get update && apt-get install -y docker-ce-cli

# Установка Ansible
RUN apt-get install -y ansible

USER jenkins

# Предустановка плагинов
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli -f /usr/share/jenkins/ref/plugins.txt
```

## plugins.txt

Создайте файл `docker/jenkins/plugins.txt` с основными плагинами:

```
ansible
docker-workflow
pipeline-aws
git
ssh-slaves
configuration-as-code
blueocean
credentials-binding
```

## Инструкция по развертыванию

1. Создайте структуру каталогов:

```bash
mkdir -p docker/jenkins
cd docker/jenkins
touch Dockerfile plugins.txt
```

2. Добавьте содержимое файлов как указано выше

3. Запустите систему:

```bash
# Создайте .env файл для переменных
echo "AGENT_SECRET=$(openssl rand -hex 16)" > .env

# Запустите систему
docker-compose up -d --build
```

4. После запуска:

- Откройте Jenkins в браузере: http://localhost:8080
- Логин: `admin`
- Пароль: `password` (или посмотрите в логах контейнера командой `docker logs corp-jenkins`)

## Решение проблем

1. **Ошибка с AGENT_SECRET**:
   - Создайте файл `.env` с переменными окружения
   - Или передайте переменные при запуске: `AGENT_SECRET=your-secret docker-compose up`

2. **Ошибка с версией docker-compose**:
   - Удалите строку `version` из файла docker-compose.yml (современные версии не требуют её указания)

3. **Ошибка с Dockerfile**:
   - Убедитесь, что Dockerfile находится в правильном пути (`docker/jenkins/Dockerfile`)
   - Проверьте права на файл

4. **Доступ к Docker сокету**:
   - Убедитесь, что у пользователя есть права на `/var/run/docker.sock`
   - Можно проверить: `ls -la /var/run/docker.sock`

Это решение должно работать без ошибок и предоставит вам Jenkins с предустановленными:
- Docker CLI
- Ansible
- Основными плагинами
- Агентами для выполнения задач
