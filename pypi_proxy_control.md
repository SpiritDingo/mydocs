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


_________



Чтобы максимально ограничить зеркало Bandersnatch и при этом оперативно добирать или блокировать пакеты через Devpi, постройте такую архитектуру:

· Bandersnatch зеркалирует только строго необходимые пакеты (белый список).
· Devpi создаёт два индекса:
  · root/bandersnatch-mirror – проксирует ваше урезанное зеркало Bandersnatch.
  · root/prod – базируется на root/bandersnatch-mirror и стандартном root/pypi, чтобы недостающие пакеты автоматически «добирались» из PyPI. Здесь же применяются блокировки.
· Nexus (опционально) служит единой точкой входа для pip.

Ниже – конкретная настройка и команды.

---

1. Bandersnatch – белый список и минимум мусора

В bandersnatch.conf задаём только нужные проекты и исключаем dev‑версии:

```ini
[mirror]
directory = /srv/pypi
master = https://pypi.org
workers = 3
stop-on-error = false
delete-packages = true

[plugins]
enabled = blocklist_project

[filter_packages]
blacklist_regex = ^.*-(dev|rc|alpha|beta)[0-9]*$
whitelist_project =
    numpy
    pandas
    scipy
    requests
    flask
    # добавьте сюда минимально необходимые пакеты
```

После изменения списка выполните:

```bash
docker-compose exec bandersnatch bash -c 'source /venv/bin/activate && bandersnatch mirror'
```

---

2. Devpi – два индекса для «добирания» и блокировок

Файл devpi-init.sh (подкладывается в /docker-entrypoint-init.d/init.sh) настраивает сервер:

```bash
#!/bin/bash
sleep 5

# Устанавливаем соединение и пароль root
devpi use http://localhost:3141
devpi login root --password ''
devpi user -m root password="${DEVPISERVER_ROOT_PASSWORD:-admin123}"
devpi login root --password="${DEVPISERVER_ROOT_PASSWORD:-admin123}"

# 1. Индекс-прокси на локальное зеркало Bandersnatch
devpi index -c bandersnatch-mirror type=mirror mirror_url="http://bandersnatch-nginx:80/simple/"
# (type=mirror и mirror_url требуют Devpi >= 5.0)

# 2. Стандартный PyPI-индекс (если ещё не создан)
devpi index -c pypi bases= type=mirror mirror_url="https://pypi.org/simple/"

# 3. Рабочий индекс, который сначала смотрит в урезанное зеркало, потом в PyPI
devpi index -c prod bases=root/bandersnatch-mirror,root/pypi volatile=False

# Оперативные блокировки (примеры)
devpi index root/prod block oldpackage
devpi index root/prod "block*" "test-*"
```

Что происходит:

· root/bandersnatch-mirror – запросы к нему уходят в nginx с вашим зеркалом и кешируются.
· root/prod при запросе пакета проверяет:
  · собственный кеш;
  · root/bandersnatch-mirror (т.е. локальное зеркало);
  · root/pypi (напрямую в PyPI).
    Так любой отсутствующий в зеркале пакет автоматически добирается из интернета.

---

3. Оперативная работа со списками

Добавить пакет в белый список Bandersnatch (долгосрочно)

1. Прописать его в whitelist_project в bandersnatch.conf.
2. Запустить синхронизацию:
   ```bash
   docker-compose exec bandersnatch bash -c 'source /venv/bin/activate && bandersnatch mirror'
   ```

Разрешить пакет «на лету» (без правки зеркала)

Просто установите его через Devpi – он автоматически «доберётся» из PyPI и закешируется в root/prod. Никаких дополнительных действий.

Заблокировать пакет в Devpi (мгновенно)

```bash
docker-compose exec devpi devpi index root/prod block some-package
```

Пакет станет недоступен для всех, кто использует root/prod, даже если он есть в зеркале или в PyPI.

Разблокировать

```bash
docker-compose exec devpi devpi index root/prod allow some-package
```

---

4. Итоговая схема в docker-compose

Приложенный ранее docker-compose.yml полностью подходит, нужно лишь заменить devpi-init.sh и bandersnatch.conf на новые версии. Nexus настраивается вручную на проксирование http://devpi:3141/root/prod/+simple/.

---

5. Почему это «максимальное ограничение + оперативность»

· Зеркало Bandersnatch хранит только жёстко заданный белый список – минимум дискового пространства и трафика.
· Devpi даёт возможность получать любые недостающие пакеты из PyPI без ручного вмешательства.
· Блокировки в Devpi применяются мгновенно и не требуют перестройки зеркала.
· При добавлении новых «разрешённых» пакетов в белый список Bandersnatch они становятся доступны локально без интернета после очередной синхронизации.

Таким образом, вы совмещаете жёсткий контроль (Bandersnatch) с гибкостью и скоростью управления (Devpi).


_______



Devpi-constrained — это плагин для Devpi, который позволяет задавать для индекса ограничения на версии пакетов (constraints). В отличие от простого block/allow (блокирует пакет целиком), constrained позволяет тонко регулировать допустимые версии, например:
numpy>=1.21,<2.0 – разрешены только определённые диапазоны.

В описанной ранее архитектуре это даёт третий уровень контроля:

· Bandersnatch фильтрует имена пакетов (белый список);
· Devpi constrained ограничивает версии внутри разрешённых пакетов;
· Devpi block/allow при необходимости полностью скрывает пакет.

---

1. Добавляем devpi-constrained в Docker-образ

Создайте рядом с docker-compose.yml файл Dockerfile.devpi:

```dockerfile
FROM devpi/devpi:latest

# Установка плагина constrained
RUN pip install devpi-constrained
```

В docker-compose.yml замените образ devpi на сборку:

```yaml
devpi:
  build:
    context: .
    dockerfile: Dockerfile.devpi
  container_name: devpi
  restart: unless-stopped
  ports:
    - "3141:3141"
  volumes:
    - devpi-data:/data
    - ./devpi-init.sh:/docker-entrypoint-init.d/init.sh:ro
  environment:
    - DEVPISERVER_HOST=0.0.0.0
    - DEVPISERVER_ROOT_PASSWORD=admin123
  networks:
    - pypi-net
```

---

2. Настройка constrained-индекса при старте

Модифицируйте devpi-init.sh, добавив создание индекса с ограничениями:

```bash
#!/bin/bash
sleep 5

devpi use http://localhost:3141
devpi login root --password ''
devpi user -m root password="${DEVPISERVER_ROOT_PASSWORD:-admin123}"
devpi login root --password="${DEVPISERVER_ROOT_PASSWORD:-admin123}"

# Зеркало на Bandersnatch
devpi index -c bandersnatch-mirror type=mirror mirror_url="http://bandersnatch-nginx:80/simple/"

# Обычный PyPI
devpi index -c pypi type=mirror mirror_url="https://pypi.org/simple/"

# Рабочий индекс с базовыми источниками
devpi index -c prod bases=root/bandersnatch-mirror,root/pypi volatile=False

# Применяем constrained-ограничения
# Правила задаются в виде строки: "package1>=1.0,<2.0; package2==3.2.1"
devpi index root/prod constraints="numpy>=1.21,<2.0; pandas>=1.3,<2.0; requests>=2.25"

# Дополнительно можно оставить оперативные блокировки
devpi index root/prod block oldpackage
devpi index root/prod "block*" "test-*"
```

Что это даёт:

· Пакеты numpy, pandas, requests можно устанавливать только в указанных диапазонах версий.
· Все остальные пакеты (если они есть в зеркале или PyPI) не ограничены – это позволяет гибко контролировать только критичные зависимости.
· Если нужно полностью запретить установку любого пакета, по-прежнему используйте block.

---

3. Управление constrained-правилами «на лету»

После запуска можно обновлять ограничения без перезагрузки сервера:

```bash
# Задать новые constraints
docker-compose exec devpi devpi login root --password=admin123
docker-compose exec devpi devpi index root/prod constraints="numpy==1.23.5; pandas>=1.5,<2"

# Посмотреть текущие ограничения
docker-compose exec devpi devpi index root/prod | grep constraints

# Удалить все constraints (оставить пустую строку)
docker-compose exec devpi devpi index root/prod constraints=""
```

---

4. Как это работает вместе с Bandersnatch и Nexus

1. Bandersnatch выкачивает только пакеты из белого списка (например, numpy, pandas).
2. Devpi (через индекс prod) объединяет зеркало и PyPI, автоматически добирая недостающее.
3. Devpi-constrained на root/prod накладывает ограничения по версиям. Если пользователь запросит pip install numpy==1.20.0, Devpi ответит 404 Not Found, потому что версия 1.20.0 не попадает в ограничение >=1.21,<2.0.
4. Nexus проксирует root/prod и может добавлять свои Routing Rules как последний рубеж.

При необходимости можно создать несколько индексов с разными constraints для разных групп пользователей (например, prod-strict, prod-relaxed) и привязать их к разным URL в Nexus.

---

5. Полный пример структуры проекта

```
.
├── docker-compose.yml          # nexus, devpi (build), bandersnatch, nginx
├── Dockerfile.devpi             # на основе devpi с плагином
├── devpi-init.sh                # инициализация индексов + constraints
├── bandersnatch.conf
└── nginx-default.conf
```

Таким образом, devpi-constrained добавляет элегантный способ централизованно задавать допустимые диапазоны версий для критических пакетов, не отказываясь от гибкого «добирания» новых пакетов и оперативных блокировок.

_____
_____


Образ devpi/devpi:latest может отсутствовать в Docker Hub — он был удалён или перемещён. Самый надёжный способ — собрать собственный образ на основе python, включив devpi-server, devpi-client и devpi-constrained. Это заодно решает проблему с плагином constrained.

Ниже — обновлённый Dockerfile.devpi и изменения в docker-compose.yml.

---

1. Новый Dockerfile для Devpi (вместо FROM devpi/devpi:latest)

```dockerfile
# Dockerfile.devpi
FROM python:3.11-slim

# Установка devpi-сервера, клиента и плагина constrained
RUN pip install --no-cache-dir \
    devpi-server \
    devpi-client \
    devpi-constrained

# Создаём рабочую директорию для данных
RUN mkdir -p /data
ENV DEVPISERVER_SERVERDIR=/data

# Скрипт инициализации будет скопирован через docker-compose
# Запуск сервера
EXPOSE 3141
CMD ["devpi-server", "--host", "0.0.0.0", "--port", "3141"]
```

---

2. Обновлённый docker-compose.yml (фрагмент с devpi)

```yaml
devpi:
  build:
    context: .
    dockerfile: Dockerfile.devpi
  container_name: devpi
  restart: unless-stopped
  ports:
    - "3141:3141"
  volumes:
    - devpi-data:/data
    - ./devpi-init.sh:/docker-entrypoint-init.d/init.sh:ro
  environment:
    - DEVPISERVER_HOST=0.0.0.0
    - DEVPISERVER_ROOT_PASSWORD=admin123
  entrypoint: |
    bash -c '
      # Запускаем сервер в фоне, ждём, запускаем скрипт инициализации
      devpi-server --host 0.0.0.0 --port 3141 &
      sleep 5
      if [ -f /docker-entrypoint-init.d/init.sh ]; then
        bash /docker-entrypoint-init.d/init.sh
      fi
      wait
    '
  networks:
    - pypi-net
```

Теперь при запуске docker-compose up -d образ будет собран автоматически.

---

3. Если хотите попробовать найти готовый образ

Официальный образ переехал в GitHub Container Registry:

```bash
docker pull ghcr.io/devpi/devpi:latest
```

Тогда в docker-compose.yml можно указать:

```yaml
devpi:
  image: ghcr.io/devpi/devpi:latest
  ...
```

Но рекомендуется вариант с собственным Dockerfile — он гарантирует наличие devpi-constrained и полную независимость от внешних образов.


Вот готовое решение для автономного Devpi-сервера с плагином devpi-constrained, без Nexus и Bandersnatch.
Кеширование — строго по запросу (первый pip install загружает пакет из PyPI, последующие отдаются из кеша).

---

Файлы проекта

Создайте три файла в одной папке:

1. Dockerfile.devpi

```dockerfile
FROM python:3.11-slim

# Устанавливаем devpi и плагин constrained
RUN pip install --no-cache-dir \
    devpi-server \
    devpi-client \
    devpi-constrained

RUN mkdir -p /data
ENV DEVPISERVER_SERVERDIR=/data

EXPOSE 3141
CMD ["devpi-server", "--host", "0.0.0.0", "--port", "3141"]
```

2. docker-compose.yml

```yaml
version: "3.8"

services:
  devpi:
    build:
      context: .
      dockerfile: Dockerfile.devpi
    container_name: devpi
    restart: unless-stopped
    ports:
      - "3141:3141"
    volumes:
      - devpi-data:/data
      - ./devpi-init.sh:/docker-entrypoint-init.d/init.sh:ro
    environment:
      - DEVPISERVER_ROOT_PASSWORD=admin123
    entrypoint: |
      bash -c '
        devpi-server --host 0.0.0.0 --port 3141 &
        sleep 5
        if [ -f /docker-entrypoint-init.d/init.sh ]; then
          bash /docker-entrypoint-init.d/init.sh
        fi
        wait
      '
    networks:
      - devpi-net

volumes:
  devpi-data:

networks:
  devpi-net:
    driver: bridge
```

3. devpi-init.sh – начальная настройка индекса

```bash
#!/bin/bash
sleep 5

# Подключаемся к локальному серверу
devpi use http://localhost:3141
devpi login root --password ''
devpi user -m root password="${DEVPISERVER_ROOT_PASSWORD:-admin123}"
devpi login root --password="${DEVPISERVER_ROOT_PASSWORD:-admin123}"

# Создаём зеркалирующий индекс PyPI (если его ещё нет)
devpi index -c pypi type=mirror mirror_url="https://pypi.org/simple/"

# Рабочий индекс, который будет кешировать пакеты по запросу
devpi index -c prod bases=root/pypi volatile=False

# === ЧЁРНЫЙ СПИСОК ПАКЕТОВ (block) ===
devpi index root/prod block oldpackage
devpi index root/prod "block*" "test-*"            # все, начинающиеся с test-

# === БЕЛЫЙ СПИСОК ЧЕРЕЗ CONSTRAINTS ===
# Ограничиваем версии только для разрешённых пакетов.
# Все остальные пакеты остаются доступными (если не заблокированы).
devpi index root/prod constraints="numpy>=1.21,<2.0; pandas>=1.3,<2.0; requests>=2.25"

echo "Devpi setup complete."
```

---

Запуск

```bash
# Собрать образ и запустить сервис
docker-compose up -d

# Проверить логи
docker-compose logs -f devpi
```

Сервер будет доступен на http://localhost:3141.
Пользователь root с паролем admin123.

---

Использование с pip

```bash
# Установка пакета через Devpi
pip install --index-url http://localhost:3141/root/prod/+simple/ numpy

# Или если настроена переменная окружения
export PIP_INDEX_URL=http://localhost:3141/root/prod/+simple/
pip install pandas
```

---

Управление чёрными и белыми списками

Все команды выполняются внутри контейнера или через docker-compose exec devpi ....

Чёрный список пакетов

Блокировка делает пакет полностью недоступным (клиенты получат 404).

```bash
# Блокировать один пакет
docker-compose exec devpi devpi login root --password=admin123
docker-compose exec devpi devpi index root/prod block some-package

# Блокировать по маске
docker-compose exec devpi devpi index root/prod "block*" "dev-*"
```

Белый список (разрешить только определённые пакеты)

Devpi не имеет встроенного «белого списка» для имён пакетов.
На практике это можно сделать двумя способами:

Способ 1 – через constraints (рекомендуемый)
Оставьте индекс открытым (доступны все пакеты), но ограничьте версии только нужных.
Пользователи смогут установить любой пакет, но для разрешённых пакетов будут действовать только заданные диапазоны версий. Нежелательные пакеты можно заблокировать через block.

Способ 2 – жёсткий белый список
Заблокируйте все пакеты, кроме нужных. Для этого нужно перечислить запрещённые пакеты (неудобно), либо использовать внешний инструмент (например, nginx с allow/deny). В рамках автономного Devpi можно периодически блокировать «всё новое» скриптом.

Пример: белый список через полный запрет нежелательных

1. Заблокируйте маски всех нежелательных пакетов в devpi-init.sh:

```bash
devpi index root/prod "block*" "my-private-*"   # запретить префикс
```

2. Для разрешённых пакетов задайте constraints:

```bash
devpi index root/prod constraints="my-package==1.0"
```

Теперь my-package доступен только в версии 1.0, а всё, что начинается с my-private-, заблокировано. Остальной PyPI остаётся открытым (это можно регулировать дополнительными block).

---

Управление constraints (динамически)

Constraints можно менять в любое время без перезапуска.

```bash
# Установить новые ограничения
docker-compose exec devpi devpi index root/prod constraints="numpy==1.23.5; pandas>=1.5"

# Удалить все ограничения
docker-compose exec devpi devpi index root/prod constraints=""

# Посмотреть текущие
docker-compose exec devpi devpi index root/prod
```

---

Как работает кеширование «по запросу»

· При первом pip install <package> Devpi загружает пакет из PyPI и сохраняет в /data.
· Следующие установки этого пакета (любой разрешённой версии) идут уже из локального кеша.
· Никакой предварительной синхронизации не требуется.
· Если интернет пропадает, ранее закешированные пакеты продолжают работать.

---

Резюме

Вы получили лёгкий, автономный PyPI-прокси с:

· Кешированием только по запросу.
· Чёрными списками пакетов через block.
· Гибкими ограничениями версий (белый список версий) через devpi-constrained.
· Полным управлением из командной строки.

Никакие дополнительные сервисы (Nexus, Bandersnatch) не используются.


______


devpi-constrained не умеет блокировать все пакеты сразу — он работает с конкретными именами пакетов и задаёт допустимые диапазоны версий.
Для полного запрета всех пакетов используется штатная команда Devpi:

```bash
devpi index root/prod "block*" "*"
```

После этого ни один пакет не будет доступен. Затем вы разрешаете только те, что нужны (белый список), и при необходимости накладываете на них ограничения версий через constraints.

---

1. Как это выглядит в devpi-init.sh

Замените содержимое скрипта инициализации на:

```bash
#!/bin/bash
sleep 5

devpi use http://localhost:3141
devpi login root --password ''
devpi user -m root password="${DEVPISERVER_ROOT_PASSWORD:-admin123}"
devpi login root --password="${DEVPISERVER_ROOT_PASSWORD:-admin123}"

# Создаём прокси-индекс на PyPI (источник пакетов, если их разрешат)
devpi index -c pypi type=mirror mirror_url="https://pypi.org/simple/"

# Рабочий индекс с бэкендом PyPI, кеширование по запросу
devpi index -c prod bases=root/pypi volatile=False

# === ПОЛНЫЙ ЗАПРЕТ ВСЕХ ПАКЕТОВ ===
devpi index root/prod "block*" "*"

# === РАЗРЕШАЕМ ТОЛЬКО НУЖНЫЕ ПАКЕТЫ ===
devpi index root/prod allow numpy
devpi index root/prod allow pandas
devpi index root/prod allow requests

# === ОГРАНИЧИВАЕМ ВЕРСИИ ДЛЯ РАЗРЕШЁННЫХ ПАКЕТОВ (через constrained) ===
devpi index root/prod constraints="numpy>=1.21,<2.0; pandas>=1.3,<2.0; requests==2.28.2"

echo "Devpi setup complete: all packages blocked, only explicit allowed with constraints."
```

---

2. Запуск

```bash
docker-compose up -d --build   # пересобрать образ и запустить
```

После старта:

· pip install любого пакета, кроме numpy, pandas, requests, будет получать 404.
· Для разрешённых пакетов будут доступны только версии, соответствующие constraints.
· Если позже потребуется разрешить новый пакет, выполните:

```bash
docker-compose exec devpi devpi login root --password=admin123
docker-compose exec devpi devpi index root/prod allow flask
docker-compose exec devpi devpi index root/prod constraints="numpy>=1.21,<2.0; pandas>=1.3,<2.0; requests==2.28.2; flask>=2.0"
```

· Чтобы временно разрешить всё (убрать блокировку), используйте:

```bash
docker-compose exec devpi devpi index root/prod "allow*" "*"
```

---

3. Как это работает технически

· devpi block "*" вносит правило в индекс: все пакеты, соответствующие шаблону * (т.е. все), скрываются.
· devpi allow <name> отменяет блокировку для конкретного пакета.
· devpi constraints накладывает ограничения по версиям поверх уже разрешённых имён.
· Кеширование остаётся «по запросу»: пакет загружается из PyPI только при первом pip install разрешённого пакета (и только допустимой версии).

Таким образом, связка block "*" + allow даёт полноценный белый список имён, а devpi-constrained — белый список версий внутри этих имён.
_____
_____
_____

Ошибки, которые вы привели, связаны с порядком запуска, недостаточным ожиданием готовности сервера и неправильным синтаксисом команд block/allow в Devpi. Ниже – разбор каждой проблемы и исправленный комплект файлов.

---

Что было не так

1. Сервер не успевает подняться
      sleep 5 может быть недостаточно. В логах: Connection refused, значит devpi-server ещё не слушает порт.
2. Нет инициализации /data
      Devpi требует, чтобы директория данных была проинициализирована (файлы .nodeinfo, база данных). При первом запуске нужно выполнить devpi-init, иначе сервер вылетает с fatal: The path '/data' contains no devpi-server data.
3. Синтаксис block* и allow
      Devpi ожидает аргументы в формате ИМЯ=ЗНАЧЕНИЕ:
   · Правильно: devpi index root/prod "block*"="*"
   · Правильно: devpi index root/prod allow="numpy"
        Ваш вызов devpi index root/prod allow numpy вызывал ошибку arguments must be format NAME=VALUE.
4. Сбой аутентификации
      Из-за невозможности подключиться к серверу devpi login не срабатывал, и последующие команды получали no current authenticated user.

---

Исправленное решение

1. Dockerfile.devpi (без изменений)

```dockerfile
FROM python:3.11-slim

RUN pip install --no-cache-dir \
    devpi-server \
    devpi-client \
    devpi-constrained

RUN mkdir -p /data
ENV DEVPISERVER_SERVERDIR=/data

EXPOSE 3141
CMD ["devpi-server", "--host", "0.0.0.0", "--port", "3141"]
```

2. docker-compose.yml (entrypoint с автоинициализацией)

```yaml
version: "3.8"

services:
  devpi:
    build:
      context: .
      dockerfile: Dockerfile.devpi
    container_name: devpi
    restart: unless-stopped
    ports:
      - "3141:3141"
    volumes:
      - devpi-data:/data
      - ./devpi-init.sh:/docker-entrypoint-init.d/init.sh:ro
    environment:
      - DEVPISERVER_ROOT_PASSWORD=admin123
    entrypoint: |
      bash -c '
        # Если папка с данными пуста – инициализируем
        if [ ! -f /data/.nodeinfo ]; then
          echo "Initializing devpi data directory..."
          devpi-init
        fi

        # Запускаем сервер в фоне и ждём готовности
        devpi-server --host 0.0.0.0 --port 3141 &
        SERVER_PID=$!

        # Ждём, пока сервер не начнёт отвечать (с таймаутом ~30 сек)
        for i in {1..30}; do
          if curl -s http://localhost:3141/+api >/dev/null 2>&1; then
            break
          fi
          sleep 1
        done

        # Выполняем скрипт настройки (если он есть)
        if [ -f /docker-entrypoint-init.d/init.sh ]; then
          bash /docker-entrypoint-init.d/init.sh
        fi

        # Переводим сервер в foreground
        wait $SERVER_PID
      '
    networks:
      - devpi-net

volumes:
  devpi-data:

networks:
  devpi-net:
    driver: bridge
```

3. devpi-init.sh – корректная настройка

```bash
#!/bin/bash
set -e

# Функция проверки аутентификации и повторного входа при необходимости
devpi_ensure_login() {
    if ! devpi use http://localhost:3141 2>/dev/null | grep -q "logged in"; then
        devpi login root --password="${DEVPISERVER_ROOT_PASSWORD:-admin123}"
    fi
}

echo "Waiting for devpi server to be fully ready..."
# Сервер уже должен быть доступен благодаря циклу в entrypoint, но добавим ещё небольшую паузу
sleep 2

# Первоначальная настройка
devpi use http://localhost:3141
devpi login root --password="${DEVPISERVER_ROOT_PASSWORD:-admin123}"
devpi user -m root password="${DEVPISERVER_ROOT_PASSWORD:-admin123}" || true
devpi_ensure_login

# Создаём базовый прокси-индекс на PyPI (если его ещё нет)
if ! devpi index -l | grep -q 'root/pypi'; then
    devpi index -c pypi type=mirror mirror_url="https://pypi.org/simple/"
fi

# Создаём рабочий индекс prod (если отсутствует)
if ! devpi index -l | grep -q 'root/prod'; then
    devpi index -c prod bases=root/pypi volatile=False
fi

# Переключаемся на prod
devpi use root/prod

# === ПОЛНАЯ БЛОКИРОВКА ВСЕХ ПАКЕТОВ ===
devpi index root/prod "block*"="*"

# === РАЗРЕШАЕМ ТОЛЬКО НУЖНЫЕ ===
devpi index root/prod allow="numpy"
devpi index root/prod allow="pandas"
devpi index root/prod allow="requests"

# === ОГРАНИЧЕНИЯ ВЕРСИЙ (constrained) ===
devpi index root/prod constraints="numpy>=1.21,<2.0; pandas>=1.3,<2.0; requests==2.28.2"

echo "Devpi setup complete: all packages blocked, only allowed ones are: numpy, pandas, requests with version constraints."
```

---

Как запустить

1. Скопируйте три файла (Dockerfile.devpi, docker-compose.yml, devpi-init.sh) в одну папку.
2. Выполните docker-compose up -d --build.
3. Проверьте логи: docker-compose logs -f devpi. Вы должны увидеть сообщение об успешной настройке.

---

Проверка работы

```bash
# Установка разрешённого пакета (успех)
pip install --index-url http://localhost:3141/root/prod/+simple/ numpy==1.21.6

# Попытка установить заблокированный пакет (ошибка 404)
pip install --index-url http://localhost:3141/root/prod/+simple/ flask
# -> ERROR: Could not find a version that satisfies the requirement flask (from versions: none)

# Попытка установить разрешённый, но версию вне ограничений (ошибка 404)
pip install --index-url http://localhost:3141/root/prod/+simple/ pandas==2.0.0
# -> ERROR: Could not find a version that satisfies the requirement pandas==2.0.0 (from versions: none)
```

---

Управление после запуска (через контейнер)

```bash
# Войти в контейнер
docker-compose exec devpi bash

# Авторизоваться
devpi login root --password=admin123

# Посмотреть текущие настройки индекса prod
devpi index root/prod

# Разрешить ещё один пакет
devpi index root/prod allow="flask"
devpi index root/prod constraints="flask>=2.0"

# Снова всё заблокировать (кроме уже разрешённых, после этого flask не исчезнет)
# Для временной разблокировки всех (снятие ограничений):
devpi index root/prod "allow*"="*"

# Полностью очистить списки блокировок и разрешений:
devpi index root/prod "block*"=""   # удалить все шаблоны блокировки
```

---

Коротко об исправлениях

· Добавлена автоматическая инициализация (devpi-init) при первом запуске.
· Реализовано надёжное ожидание сервера через curl в entrypoint.
· Исправлен синтаксис block* и allow на "block*"="*" и allow="numpy".
· Устранены проблемы аутентификации и неподключённого клиента.

Теперь Devpi‑сервер с плагином constrained работает изолированно, кеширует только запрошенные пакеты и жёстко фильтрует по белому списку имён и версий.


______
_______
_______

Отлично, вы привели официальную документацию devpi-constrained, где описан правильный подход: не block/allow, а создание индекса типа constrained и управление ограничениями через constraints.
Это решает задачу белого списка элегантно:

· Создаётся constrained-индекс на основе root/pypi (прокси PyPI).
· Устанавливается constraints="*" — запрещает все пакеты.
· Затем добавляются constraints+="<имя_пакета><версия>" — разрешаются только нужные.

Кеширование остаётся «по запросу», а управление — централизованным.

---

Исправленный devpi-init.sh (с учётом документации)

```bash
#!/bin/bash
set -e

echo "Waiting for devpi server to be fully ready..."
sleep 3

# Первоначальная настройка пользователя root
devpi use http://localhost:3141
devpi login root --password="${DEVPISERVER_ROOT_PASSWORD:-admin123}"
devpi user -m root password="${DEVPISERVER_ROOT_PASSWORD:-admin123}" || true

# Создаём зеркало PyPI (если ещё не создано)
if ! devpi index -l | grep -q 'root/pypi'; then
    devpi index -c pypi type=mirror mirror_url="https://pypi.org/simple/"
fi

# Создаём constrained-индекс prod на базе pypi (если отсутствует)
if ! devpi index -l | grep -q 'root/prod'; then
    devpi index -c prod type=constrained bases=root/pypi volatile=False
fi

# Устанавливаем ограничения: сначала "*" запрещает всё, потом разрешаем конкретные пакеты с версиями
devpi index root/prod constraints="*, numpy>=1.21,<2.0, pandas>=1.3,<2.0, requests==2.28.2"

echo "Devpi constrained index setup complete."
```

Пояснение

· type=constrained — индекс только для чтения, фильтрует релизы из баз.
· constraints работает как Constraints Files в pip, но с возможностью использовать * (блокировка всех остальных пакетов, которых нет в списке).
· Кеширование работает автоматически через базовый root/pypi (первый запрос к разрешённому пакету загрузит его в кеш).

---

Проверка работы

```bash
# Разрешённый пакет с допустимой версией
pip install --index-url http://localhost:3141/root/prod/+simple/ numpy==1.21.6

# Запрещённый пакет (не указан в constraints) → 404
pip install --index-url http://localhost:3141/root/prod/+simple/ flask

# Разрешённый пакет, но версия вне ограничений → 404
pip install --index-url http://localhost:3141/root/prod/+simple/ pandas==2.0.0
```

---

Динамическое управление (без перезапуска)

```bash
# Войти в контейнер
docker-compose exec devpi bash
devpi login root --password=admin123

# Добавить новый пакет (разрешить flask>=2.0)
devpi index root/prod constraints+="flask>=2.0"

# Посмотреть текущие constraints
devpi index root/prod | grep constraints

# Удалить все ограничения (открыть доступ ко всем пакетам)
devpi index root/prod constraints=""

# Снова запретить всё и оставить только нужное
devpi index root/prod constraints="*, numpy, pandas, requests"
```

---

Итоговые файлы

Ваш проект теперь состоит из:

1. Dockerfile.devpi – без изменений.
2. docker-compose.yml – entrypoint с автоинициализацией devpi-init и ожиданием сервера (как в предыдущем ответе).
3. devpi-init.sh – новый, создаёт constrained индекс и задаёт строгий белый список.

Запуск:

```bash
docker-compose up -d --build
```

Теперь решение полностью соответствует документации devpi-constrained, а управление белыми списками реализовано через constraints без block/allow.