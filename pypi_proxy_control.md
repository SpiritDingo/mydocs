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