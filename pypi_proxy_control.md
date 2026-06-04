Ниже — готовый docker-compose.yml, который поднимает:

· Sonatype Nexus 3 (прокси PyPI с возможностью Routing Rules);
· Devpi (прокси PyPI с блокировкой/разрешением пакетов);
· Bandersnatch (локальное зеркало PyPI с фильтрацией пакетов) + nginx для раздачи зеркала.

Все сервисы связаны через общую сеть, данные хранятся в именованных томах, а настройки чёрных/белых списков задаются на уровнях Devpi и Bandersnatch. Nexus настраивается через веб-интерфейс после первого запуска.

```yaml
# docker-compose.yml
version: "3.8"

services:
  # ================== Nexus 3 ==================
  nexus:
    image: sonatype/nexus3:latest
    container_name: nexus
    restart: unless-stopped
    ports:
      - "8081:8081"   # Web UI + API
    volumes:
      - nexus-data:/nexus-data
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms512m -Xmx512m -XX:MaxDirectMemorySize=256m
    networks:
      - pypi-net

  # ================== Devpi ==================
  devpi:
    image: devpi/devpi:latest
    container_name: devpi
    restart: unless-stopped
    ports:
      - "3141:3141"
    volumes:
      - devpi-data:/data
      - ./devpi-init.sh:/docker-entrypoint-init.d/init.sh:ro  # автонастройка индекса
    environment:
      - DEVPISERVER_HOST=0.0.0.0
      - DEVPISERVER_ROOT_PASSWORD=admin123  # пароль root (см. скрипт)
    networks:
      - pypi-net

  # ================== Bandersnatch (зеркалирование) ==================
  bandersnatch:
    image: python:3.11-slim  # лёгкий образ, в который доустановим bandersnatch
    container_name: bandersnatch
    restart: unless-stopped
    volumes:
      - bandersnatch-mirror:/srv/pypi            # зеркало (общий том с nginx)
      - ./bandersnatch.conf:/etc/bandersnatch.conf:ro
      - bandersnatch-venv:/venv
    entrypoint: |
      bash -c '
        if [ ! -d /venv/bin ]; then
          python -m venv /venv
        fi
        source /venv/bin/activate
        pip install -q bandersnatch
        echo "Initial mirror sync..."
        bandersnatch mirror
        echo "Sync done. Refreshing every 12 hours."
        while true; do
          sleep 43200
          bandersnatch mirror
        done
      '
    networks:
      - pypi-net

  # ================== Nginx для раздачи зеркала ==================
  bandersnatch-nginx:
    image: nginx:alpine
    container_name: bandersnatch-nginx
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - bandersnatch-mirror:/usr/share/nginx/html/web:ro
      - ./nginx-default.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - pypi-net

volumes:
  nexus-data:
  devpi-data:
  bandersnatch-mirror:
  bandersnatch-venv:

networks:
  pypi-net:
    driver: bridge
```

---

Вспомогательные файлы

1. devpi-init.sh – скрипт, который выполнится при первом запуске Devpi, создаст пользователя root с паролем и индекс root/prod. При необходимости тут же блокируются пакеты.

```bash
#!/bin/bash
# Ждём готовности сервера
sleep 5
devpi use http://localhost:3141
devpi login root --password ''
devpi user -m root password="${DEVPISERVER_ROOT_PASSWORD:-admin123}"
devpi login root --password="${DEVPISERVER_ROOT_PASSWORD:-admin123}"
devpi index -c prod bases=root/pypi volatile=False
# Пример чёрного списка – блокируем пакет oldpackage и всё, что начинается с "test-"
devpi index root/prod block oldpackage
devpi index root/prod "block*" "test-*"
# Пример белого списка не требуется, но можно разблокировать явно:
# devpi index root/prod allow numpy
```

2. bandersnatch.conf – конфигурация для Bandersnatch с фильтрацией пакетов.

```ini
[mirror]
directory = /srv/pypi
master = https://pypi.org
workers = 3
stop-on-error = false
delete-packages = true

[plugins]
enabled = blocklist_project

# Фильтры: чёрный список по регулярному выражению и белый список проектов
[filter_packages]
blacklist_regex = ^.*-(dev|rc|alpha|beta)[0-9]*$
whitelist_project =
    numpy
    pandas
    scipy
    requests
    flask
```

3. nginx-default.conf – конфиг для Nginx, раздающего зеркало.

```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        root /usr/share/nginx/html;
        autoindex on;
        # убираем trailing slash для простых запросов
        rewrite ^/simple/$ /simple/index.html break;
    }
}
```

---

Запуск и начальная настройка

```bash
# 1. Клонируйте/создайте папку с тремя файлами: docker-compose.yml, devpi-init.sh, bandersnatch.conf, nginx-default.conf
# 2. Дайте права на выполнение скрипту (если из Linux):
chmod +x devpi-init.sh
# 3. Запустите стэк:
docker-compose up -d
```

После запуска:

· Devpi доступен на http://localhost:3141 (пользователь root, пароль admin123). Прокси-индекс root/prod уже настроен.
· Bandersnatch синхронизируется в течение нескольких минут, после чего зеркало будет доступно через nginx на http://localhost:8080/simple/.
· Nexus – на http://localhost:8081. Первоначальный пароль администратора лежит внутри контейнера (docker exec nexus cat /nexus-data/admin.password).

---

Интеграция в Nexus (ручная настройка)

1. Войдите в Nexus под учётной записью admin.
2. Создайте прокси-репозитории PyPI:
   · pypi-devpi-proxy: Remote storage = http://devpi:3141/root/prod/+simple/
   · pypi-bandersnatch-proxy: Remote storage = http://bandersnatch-nginx:80/simple/
     (Можно использовать только один, по вашему выбору.)
3. Создайте hosted-репозиторий pypi-hosted (для ваших приватных пакетов).
4. Создайте группу pypi-group, включив в неё pypi-hosted, pypi-devpi-proxy (или pypi-bandersnatch-proxy).
5. Настройте Routing Rules (чёрные/белые списки на уровне Nexus):
   · Перейдите в Administration → Repository → Routing Rules.
   · Создайте правило, например, с именем block-oldpackage, Matcher = /packages/oldpackage/.*, Mode = BLOCK.
   · Примените это правило к нужному прокси-репозиторию в его настройках (вкладка Routing Rule).

Теперь ваши клиенты pip могут устанавливать пакеты через группу:

```bash
pip install --index-url http://localhost:8081/repository/pypi-group/simple/ <package>
```

---

Где управлять списками

Уровень Инструмент Как задать
Зеркало Bandersnatch bandersnatch.conf → [filter_packages] (чёрный/белый списки пакетов, которые попадут в зеркало)
Прокси-кеш Devpi Команды devpi index root/prod block <pkg> / allow <pkg> (мгновенно, без перезагрузки)
Прокси-сервер Nexus Routing Rules (регулярки по URL, перехватывают запросы к /packages/...)

Рекомендуется сначала максимально ограничить зеркало через Bandersnatch, затем оперативно добирать/блокировать пакеты в Devpi, а Nexus использовать как центральную точку входа и финальный фильтр.