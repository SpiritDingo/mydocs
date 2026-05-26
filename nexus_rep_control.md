В Sonatype Nexus Repository Manager **нет встроенной функции квот или лимитов на количество/скорость скачиваний** (rate limiting, bandwidth throttling, daily quotas). Однако в Nexus реализован мощный набор механизмов для **ограничения доступа к артефактам**, фильтрации запросов и управления тем, *кто*, *что* и *откуда* может скачивать.

Ниже приведены все доступные в Nexus 3.x механизмы ограничения скачивания и конкретные примеры для каждого типа репозитория.

---
## 🔑 Основные механизмы ограничения в Nexus
| Механизм | Назначение | Работает с |
|----------|------------|------------|
| **Routing Rules** | Блокировка/пропуск запросов по IP, пути, группе, версии | `proxy`, `group` |
| **Content Selectors** | Гибкие фильтры артефактов (формат, путь, метаданные, атрибуты) | Все типы |
| **Privileges & Roles** | Разрешение на `read`/`browse`/`download` для конкретных репозиториев | Все типы |
| **Внешний Reverse Proxy** (Nginx/Apache/Traefik) | Rate limiting, IP-фильтры, bandwidth control | Все типы (проксируется к Nexus) |

> ⚠️ `Routing Rules` **не применяются к `hosted` репозиториям** напрямую. Для них используются `Content Selectors` + `Privileges`.

---
## 📦 1. Proxy репозиторий (кэш удалённых репозиториев)
**Задача:** Запретить скачивание пакетов из определённых групп/версий, ограничить доступ по IP.

### Пример A: Блокировка артефактов по пути/группе (Routing Rule)
1. `Configuration → Repository → Routing Rules → Create`
2. Заполните поля:
   - **Rule Name:** `block-apache-commons`
   - **Description:** `Block all org.apache.commons artifacts`
   - **Action:** `Deny`
   - **Condition Type:** `Path Regex`
   - **Match:** `^.*org/apache/commons/.*$`
   - **Repositories:** выберите ваш `proxy` репозиторий
3. `Save`

### Пример B: Ограничение по IP-адресам
1. В том же `Routing Rules` создайте правило:
   - **Action:** `Deny`
   - **Condition Type:** `IP Address`
   - **Match:** `192.168.50.0/24`
   - **Repositories:** ваш `proxy`
2. Все запросы из этой подсети получат `403 Forbidden`.

### Пример C: Комбинация с Privilege
- Создайте Privilege: `nx-repository-view-maven2-my-proxy-read`
- Назначьте её только роли `dev-team`
- Остальные пользователи не увидят репозиторий и не смогут скачивать через него.

---
## 📁 2. Hosted репозиторий (внутренние артефакты)
**Задача:** Разрешить скачивание только определённым командам, скрыть sensitive-пакеты.

### Пример A: Ограничение через Content Selector + Privilege
1. `Security → Content Selectors → Create`
   - **Name:** `internal-restricted`
   - **Format:** `maven2`
   - **Expression:** 
     ```groovy
     path =~ "^/com/company/internal/.*" && attributes.security.restricted == "true"
     ```
   *(Атрибут `restricted` можно выставить при деплое или через REST API)*
2. `Security → Privileges → Create → Content Selector Privilege`
   - **Name:** `read-internal-restricted`
   - **Repository:** ваш `hosted`
   - **Content Selector:** `internal-restricted`
   - **Actions:** `Read`, `Browse`
3. Назначьте привилегию только роли `core-team`.

### Пример B: Блокировка скачивания snapshot-версий для внешних потребителей
1. Content Selector:
   ```groovy
   format == "maven2" && path =~ "^.*-SNAPSHOT.*"
   ```
2. Привилегия с `Deny` (в Nexus OSS нет явного `Deny` в привилегиях, поэтому:
   - Создайте роль `external-users`
   - **НЕ** давайте ей `read` для данного селектора
   - Или используйте `Routing Rule` на `group`, куда входит этот hosted)

---
## 🔗 3. Group репозиторий (агрегатор)
**Задача:** Объединить несколько репозиториев, но ограничить доступ к отдельным членам группы.

### Особенности
- `Group` не хранит данные, а маршрутизирует запросы к членам.
- Ограничения применяются **к каждому члену отдельно**, но можно добавить `Routing Rule` на сам `group`.

### Пример: Скрыть внутренний `hosted` от внешних клиентов
1. В `group` добавьте членами: `maven-central-proxy`, `internal-releases`
2. Создайте `Routing Rule` для `group`:
   - **Action:** `Deny`
   - **Condition Type:** `Path Regex`
   - **Match:** `^/com/company/internal/.*$`
   - **Repositories:** ваш `group`
3. Теперь даже если `internal-releases` разрешён для чтения, запросы через `group` будут блокироваться.
4. Альтернатива: создайте **два разных `group`**:
   - `maven-group-internal` (с `internal-releases`)
   - `maven-group-external` (без него)
   И раздавайте разные URL разным командам.

---
## 🌐 Внешнее ограничение через Nginx (рекомендуется для rate-limit / bandwidth)
Так как Nexus OSS не ограничивает скорость/количество запросов, ставьте перед ним Nginx:

```nginx
http {
    limit_req_zone $binary_remote_addr zone=nexus:10m rate=5r/s;
    limit_conn_zone $binary_remote_addr zone=conn_nexus:10m;

    server {
        listen 8080;
        server_name nexus.company.local;

        location /repository/ {
            limit_req zone=nexus burst=10 nodelay;
            limit_conn conn_nexus 20;

            # Опционально: ограничение скорости отдачи
            # limit_rate 500k;

            proxy_pass http://127.0.0.1:8081;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```
- `rate=5r/s` → не более 5 запросов в секунду с одного IP
- `burst=10` → очередь из 10 запросов
- `limit_conn` → макс. одновременных подключений
- Работает прозрачно для Nexus, защищает от DDoS и злоупотреблений

---
## ✅ Проверка конфигурации
```bash
# Проверка доступа
curl -I -u user:pass https://nexus.company.local/repository/maven-proxy/org/apache/commons/commons-lang3/3.12.0/commons-lang3-3.12.0.jar

# Ожидается:
# HTTP/1.1 200 OK  (если разрешено)
# HTTP/1.1 403 Forbidden (если сработало Routing Rule или нет Privilege)
# HTTP/1.1 404 Not Found (если артефакт отсутствует в кэше)
```

В UI Nexus: `Browse → Repository → ваш репозиторий → Search` должен показывать/скрывать артефакты в зависимости от прав текущего пользователя.

---
## ⚠️ Важные нюансы
1. **Routing Rules не кэшируются мгновенно.** После изменения может потребоваться `Purge unused` или перезагрузка прокси-кэша.
2. **Content Selectors** доступны с Nexus 3.21+. В старых версиях используйте только `Privileges` по репозиторию.
3. **Nexus Pro** добавляет:
   - Repository Health Check (аналитика загрузок)
   - Advanced rate limiting & IP filtering
   - Audit log с детализацией скачиваний
4. **Hosted + Routing Rules:** Nexus игнорирует routing rules для hosted. Используйте `Content Selectors` + `Roles`.
5. **Docker/PyPI/npm:** Механизмы аналогичны, но синтаксис `path` отличается. Например, для npm: `path =~ "^/lodash/.*"`

---
## 📌 Итог
| Тип репозитория | Рекомендуемый механизм |
|-----------------|------------------------|
| `proxy` | `Routing Rules` (IP/path/group) + `Privileges` |
| `hosted` | `Content Selectors` + `Privileges/Roles` |
| `group` | `Routing Rules` на группе + контроль прав у членов |
| Скорость/кол-во загрузок | Nginx/Traefik `limit_req` / `limit_rate` |

Если уточните ваш формат (Maven, npm, Docker, PyPI и т.д.) и конкретную цель (например, "запретить скачивание SNAPSHOT извне" или "ограничить 100 загрузок/день"), я подготовлю точный конфиг под ваш стек.

_________


В Nexus Routing Rules применяются к **полному пути запроса** (обычно начинается с `/repository/<имя-proxy>/`). Ниже приведены готовые примеры регулярных выражений и настройки для каждого формата. Все примеры предполагают создание правила с `Action: Deny` и привязкой к конкретному proxy-репозиторию.

---
### 🐳 Docker Proxy
**Особенность:** Docker API использует пути `/v2/<namespace>/<repo>/manifests/<tag>` и `/v2/.../blobs/<digest>`. Блокировка по тегу (например, `:latest`) ненадёжна, так как клиент сначала запрашивает digest. Рекомендуется блокировать по имени образа/namespace.

| Цель | Path Regex | Комментарий |
|------|------------|-------------|
| Блокировать все образы из `untrusted-org` | `^/repository/docker-proxy/v2/untrusted-org/.*$` | Запретит pull и push |
| Блокировать конкретный образ `malware-image` | `^/repository/docker-proxy/v2/.*/malware-image/.*$` | Работает для любых namespace |
| Блокировать `latest` теги (ограниченно) | `^/repository/docker-proxy/v2/.*/manifests/latest$` | Блокирует только запрос манифеста, blobs могут закэшироваться ранее |

**UI:** `Configuration → Repository → Routing Rules → Create` → Action: `Deny`, Condition: `Path Regex`, Repository: ваш `docker-proxy`

---
### 🐍 PyPI Proxy
**Особенность:** Клиенты сначала обращаются к `/simple/<package>/` (индекс), затем скачивают файлы из `/packages/<hash>/<filename>`. Блокировка `/simple/` полностью скрывает пакет от `pip`.

| Цель | Path Regex | Комментарий |
|------|------------|-------------|
| Запретить пакет `restricted-pkg` | `^/repository/pypi-proxy/simple/restricted-pkg/.*$` | `pip` не найдёт пакет |
| Запретить все pre-release (`-alpha`, `-rc`, `-beta`) | `^/repository/pypi-proxy/packages/.*[-_](alpha|beta|rc)\d.*\.whl$` | Блокирует только колеса, не sdist |
| Блокировать домен `pypi.org` полностью | `^/repository/pypi-proxy/.+` | ⚠️ Радикально, используйте только для изоляции |

**Совет:** Для PyPI эффективнее комбинировать Routing Rule + Content Selector, если нужно блокировать по версиям или атрибутам.

---
### 📦 Debian (apt) Proxy
**Особенность:** `apt` скачивает метаданные из `/dists/<suite>/...` и пакеты из `/pool/<component>/...`. Блокировка `/dists/` сломает `apt-get update`. Безопасно блокировать только `/pool/`.

| Цель | Path Regex | Комментарий |
|------|------------|-------------|
| Запретить компонент `non-free` | `^/repository/deb-proxy/pool/non-free/.*$` | Пакеты не скачаются, метаданные останутся |
| Запретить конкретный пакет `unwanted-app` | `^/repository/deb-proxy/pool/.*unwanted-app.*\.deb$` | Точная блокировка по имени `.deb` |
| Разрешить только `main` | `^(?!.*pool/(contrib|non-free)/).*` → `Action: Allow`<br>`^/repository/deb-proxy/pool/(contrib|non-free)/.*$` → `Action: Deny` | Двухступенчатое правило |

**Важно:** После блокировки пакетов запустите `apt-get update` на клиенте. Кэшированные `.deb` в Nexus останутся, но новые скачивания будут `403`.

---
### 📦 RPM (Yum/DNF) Proxy
**Особенность:** Yum сначала загружает `/repodata/repomd.xml`, где перечислены все пакеты. Блокировка `/repodata/` сломает репозиторий. Блокировать нужно `/packages/` или конкретные пути в `repodata`.

| Цель | Path Regex | Комментарий |
|------|------------|-------------|
| Запретить пакет `restricted-rpm` | `^/repository/rpm-proxy/packages/.*/restricted-rpm-.*\.rpm$` | Безопасно для метаданных |
| Запретить группу `debuginfo` | `^/repository/rpm-proxy/packages/.*/debuginfo/.*$` | Часто занимает много места |
| Блокировать всё из `untrusted-repo` | `^/repository/rpm-proxy/repodata/.*untrusted.*$` | ⚠️ Может вызвать `404` при `yum makecache`, используйте осторожно |

**Рекомендация:** Для RPM лучше использовать `yum/dnf` на стороне клиента с `exclude=` или `yum-plugin-versionlock`, так как Yum-клиенты кэшируют метаданные агрессивно.

---
## 🔍 Как отлаживать и применять правила
1. **Точный путь запроса:** Включите DEBUG-лог для routing:
   ```xml
   <!-- В logback-nexus.xml -->
   <logger name="org.sonatype.nexus.repository.routing" level="DEBUG"/>
   ```
   В `nexus.log` увидите: `Routing rule evaluation: path='/repository/docker-proxy/v2/...'`
2. **Порядок правил:** Nexus применяет **все** совпадающие правила. Если хоть одно `Deny` совпало → `403 Forbidden`. Ставьте `Deny` выше `Allow` для предсказуемости.
3. **Очистка кэша после блокировки:** Заблокированные артефакты остаются в кэше. Чтобы удалить их:
   - `Browse → Repository → ваш proxy → Browse`
   - Найдите артефакт → `Delete`
   - Или REST API: `DELETE /service/rest/v1/components?repository=docker-proxy&name=...`

---
## ⚠️ Критические ограничения Nexus
| Формат | Ограничение Routing Rules | Обходной путь |
|--------|---------------------------|---------------|
| **Docker** | Не блокирует `docker pull` по digest, если blob уже в кэше | Блокировать на уровне registry-прокси или использовать Nginx `deny` |
| **PyPI** | `pip` кеширует wheels локально | Добавьте `--no-cache-dir` или используйте `pip.conf` с `trusted-host` ограничениями |
| **Debian/RPM** | Методы `apt update` / `yum makecache` запрашивают `/dists/` или `/repodata/` | Не блокируйте метаданные, только `/pool/` или `/packages/` |
| **Все** | Нет квот/скорости/лимитов загрузок | Nginx `limit_req`, `limit_rate`, `auth_request` или Nexus Pro |

---
## ✅ Готовый чек-лист настройки
1. `Configuration → Repository → Routing Rules → Create`
2. `Action: Deny`
3. `Condition Type: Path Regex`
4. Вставьте regex из таблицы выше (замените `<имя-proxy>` на реальное имя)
5. В поле `Repositories` выберите **только** нужный proxy
6. `Save` → протестируйте `curl -I https://nexus/repository/<proxy>/...`
7. При необходимости очистите кэш заблокированных артефактов

Если нужно заблокировать скачивание **по ролям** (например, `dev` может тянуть из PyPI, но не из Docker), создайте `Privilege: nx-repository-view-docker-<name>-read` и назначьте её только нужной роли. Routing Rules работают глобально, Privileges – по пользователям/группам.

Уточните, если нужна интеграция с CI/CD (GitLab/Jenkins) или настройка зеркал с фильтрацией по версиям.

________________
________________

Ниже приведены **углублённые, production-ready примеры** настройки ограничений для `proxy` репозиториев в Nexus 3.x. Все примеры учитывают реальную структуру API каждого формата, особенности кэширования клиентов и поведение Java-Regex в Nexus.

---
## 📐 Базовые принципы работы Routing Rules в Nexus
| Правило | Поведение |
|---------|-----------|
| **Path Regex** | Применяется к полному HTTP-пути: `/repository/<имя-proxy>/...` |
| **Приоритет** | Если срабатывает **хотя бы одно `Deny`** → `403 Forbidden`. `Allow` игнорируется. |
| **Кэш** | Правило не удаляет уже закэшированные артефакты. Требуется ручной/REST purge. |
| **Формат Regex** | Используется Java `java.util.regex`. Экранируйте точки: `\.` |

> 💡 **Совет:** В UI привязывайте правило к конкретному репозиторию. В regex пишите путь **без** префикса `/repository/<имя>/`, если правило уже привязано. Если не привязано → используйте полный путь.

---
## 🐳 Docker Proxy (Deep Dive)
Docker Registry API v2 имеет чёткую структуру. Клиент сначала запрашивает манифест по тегу, получает `digest`, затем скачивает слои (blobs).

### 🔹 Пример 1: Блокировка namespace/организации
```regex
^/v2/untrusted-org/.*
```
**Что блокирует:** `docker pull untrusted-org/app:latest`, `docker push`, запросы к каталогу.
**Почему работает:** Все запросы к образам этой организации начинаются с `/v2/untrusted-org/`.

### 🔹 Пример 2: Блокировка конкретных образов в любом namespace
```regex
^/v2/(library|mycompany)/malware-app(/.*)?$
```
**Что блокирует:** `docker pull malware-app`, `docker pull mycompany/malware-app:v1.2`

### 🔹 Пример 3: Блокировка тегов `:latest`, `:nightly`, `:dev`
```regex
^/v2/.*/manifests/(latest|nightly|dev|latest.*)$
```
⚠️ **Важно:** Блокируется только запрос манифеста. Если blob уже в кэше, `docker pull` может завершиться частично. Для полной блокировки добавьте:
```regex
^/v2/.*/blobs/sha256:.*
```
*(Блокирует все слои → радикально, используйте только для изоляции)*

### 🔹 Тестирование
```bash
# Должно вернуть 403
curl -I https://nexus.company.com/repository/docker-proxy/v2/untrusted-org/app/manifests/latest

# Docker клиент (покажет "denied: requested access to the resource is denied")
docker pull nexus.company.com/docker-proxy/untrusted-org/app:latest
```

---
## 🐍 PyPI Proxy (Deep Dive)
`pip` использует два пути: `/simple/<project>/` (HTML-индекс) и `/packages/<hash>/<filename>` (файлы).

### 🔹 Пример 1: Полное скрытие пакета от `pip`
```regex
^/simple/restricted-lib(/.*)?$
```
**Результат:** `pip search` и `pip install restricted-lib` → `ERROR: Could not find a version that satisfies the requirement`

### 🔹 Пример 2: Блокировка конкретных версий (Java Regex)
```regex
^/packages/.*/restricted_lib-(1\.(0|1)\.\d+|2\.0\.[0-5])(-.*|)\.(whl|tar\.gz|zip)$
```
**Разбор:**
- `1.0.x`, `1.1.x`, `2.0.0`..`2.0.5` → заблокированы
- `2.0.6`, `2.1.0` → пропустятся

### 🔹 Пример 3: Глобальный запрет pre-release
```regex
^/packages/.*/[^/]*[_.](a|b|c|rc|alpha|beta|dev)\d.*\.(whl|tar\.gz|zip)$
```
**Особенность:** Соответствует PEP 440. Блокирует `package-1.0a1`, `lib_2.0rc3`, `tool-dev1`.

### 🔹 Тестирование
```bash
# pip (обязательно отключите кэш, иначе pip возьмёт локальную копию)
PIP_NO_CACHE_DIR=1 pip install --index-url https://nexus.company.com/repository/pypi-proxy/simple/ restricted-lib
```

---
## 📦 Debian (apt) Proxy (Deep Dive)
Структура: `/dists/<suite>/` (метаданные), `/pool/<component>/` (пакеты). **Никогда не блокируйте `/dists/`!**

### 🔹 Пример 1: Блокировка компонентов `non-free` и `contrib`
```regex
^/pool/(non-free|contrib)/.*
```
**Результат:** `apt update` проходит, но `apt install <non-free-pkg>` → `403 Forbidden`

### 🔹 Пример 2: Блокировка пакета по имени (любые версии/архитектуры)
```regex
^/pool/.*/(unwanted-tool)-[0-9.]+.*\.deb$
```
**Примечание:** Дефис перед версией обязателен в Debian naming convention.

### 🔹 Пример 3: Разрешить только `main` (Allow/Deny цепочка)
1. Правило 1 (Allow): `^/pool/main/.*` → Action: `Allow`
2. Правило 2 (Deny): `^/pool/.*` → Action: `Deny`
**Логика Nexus:** Если сработало `Allow` и **нет** `Deny` → пропуск. Если сработал `Deny` → блок, даже если было `Allow`. Поэтому порядок важен: ставьте `Deny` ниже `Allow`.

### 🔹 Тестирование
```bash
# После изменения правил обновите кэш
apt-get update
apt-get install unwanted-tool  # Ожидается 403 или "Unable to fetch"
```

---
## 📦 RPM (Yum/DNF) Proxy (Deep Dive)
Структура: `/repodata/` (метаданные), `/packages/<arch>/<pkg>.rpm`. **Не блокируйте `/repodata/`!**

### 🔹 Пример 1: Блокировка debuginfo и src-пакетов
```regex
^/packages/.*/(debug|src)/.*\.rpm$
```
**Зачем:** Экономия дискового пространства на сервере и трафика у клиентов.

### 🔹 Пример 2: Блокировка конкретного пакета
```regex
^/packages/.*/unwanted-svc-[0-9].*\.rpm$
```
**Нюанс:** Если пакет указан в зависимостях другого RPM, `yum install` упадёт с `Error: Package: unwanted-svc`. Это ожидаемое поведение.

### 🔹 Пример 3: Разрешить только `x86_64`
```regex
^/packages/((?!x86_64).)*\.rpm$
```
**Как работает:** Negative lookahead запрещает любые пути, где **нет** `x86_64` перед `.rpm`.

### 🔹 Тестирование
```bash
yum clean all
yum makecache
yum install unwanted-svc  # 403 Forbidden или "No package available"
```

---
## 🔀 Продвинутые паттерны и комбинации

### 🧩 1. IP + Path (Требует Nexus Pro)
В Pro-версии можно создать правило:
- **Condition Type:** `IP Address`
- **Match:** `10.0.50.0/24`
- **Action:** `Deny`
- **Repositories:** `pypi-proxy`
В OSS это делается через Nginx: `deny 10.0.50.0/24;` в location `/repository/pypi-proxy/`.

### 🧩 2. HTTP Method Filtering (Pro)
Разрешить `HEAD`/`GET` для проверки существования, но запретить скачивание:
- **Condition Type:** `HTTP Method`
- **Match:** `GET`
- **Action:** `Deny`
*(В OSS метод не фильтруется, только путь)*

### 🧩 3. Комбинация с Content Selectors (для Hosted/Group)
Routing Rules не работают с `hosted`. Для него:
1. Content Selector: `path =~ "^/internal/.*" && format == "rpm"`
2. Privilege: `nx-repository-view-rpm-internal-read` → привязать к селектору
3. Назначить привилегию только роли `ci-pipeline`

---
## 🛠️ Отладка и управление кэшем

### 🔍 Включить DEBUG логирование правил
```bash
# В UI: Administration → System → Logging → Add Logger
Name: org.sonatype.nexus.repository.routing.internal
Level: DEBUG
```
В `nexus.log` появится:
```
DEBUG [qtp123-456] org.sonatype.nexus.repository.routing.internal.RoutingRuleServiceImpl - 
Routing rule evaluation: path='/repository/pypi-proxy/simple/restricted-lib/', 
rules=[Rule{name='block-restricted', action=Deny, regex='^/simple/restricted-lib/.*'}]
```

### 🗑️ Очистка закэшированных заблокированных артефактов
Routing Rules не удаляют кэш. Очистите вручную:
```bash
# 1. Поиск компонентов
curl -u admin:admin123 -X GET "https://nexus/service/rest/v1/search?repository=docker-proxy&docker.imageName=untrusted-org/app"

# 2. Удаление по ID
curl -u admin:admin123 -X DELETE "https://nexus/service/rest/v1/search?repository=docker-proxy&docker.imageName=untrusted-org/app"

# Или через UI: Browse → Repository → docker-proxy → найти → Delete
```

### 🔄 Автоматический purge (REST API)
Для CI/CD удобно удалять кэш после добавления правила:
```bash
curl -u admin:admin123 -X POST \
  "https://nexus/service/rest/v1/script" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "purge-blocked",
    "type": "groovy",
    "content": "repository.cleanup(repository.manager.get(\"docker-proxy\"), [\"docker.cleanup\"])"
  }'
```

---
## ⚠️ Почему правило может НЕ работать?

| Симптом | Причина | Решение |
|---------|---------|---------|
| `200 OK` вместо `403` | Артефакт уже в кэше | Сделайте `DELETE` или `Purge unused` |
| `pip/apt/yum` берёт старую версию | Кэш клиента | `pip install --no-cache-dir`, `apt-get clean`, `yum clean all` |
| Правило игнорируется | Regex не совпадает с полным путём | Проверьте в логах точный `path=` |
| `Allow` не срабатывает | Есть другое `Deny`, которое тоже совпало | Удалите или уточните regex conflicting rule |
| Docker pull работает частично | Блокируется только `manifests`, но `blobs` уже в кэше | Добавьте правило на `/v2/.*/blobs/.*` или очистите кэш |

---
## 📡 Массовое управление правилами через REST API
```json
// POST /service/rest/v1/routing-rules
{
  "name": "block-untrusted-pypi",
  "description": "Deny restricted packages",
  "mode": "Deny",
  "matchers": [
    {
      "type": "path",
      "pattern": "^/repository/pypi-proxy/simple/(restricted-lib|malware-pkg)/.*"
    }
  ],
  "repositories": ["pypi-proxy"]
}
```
**GET/PUT/DELETE:** `/service/rest/v1/routing-rules/{ruleName}`

---
## ✅ Итоговая матрица применимости

| Формат | Безопасно блокировать | Никогда не блокировать | Клиентский кэш |
|--------|----------------------|------------------------|----------------|
| **Docker** | `/v2/<org>/`, `/v2/.../manifests/<tag>` | `/v2/_catalog`, `/v2/token` | `docker image prune` |
| **PyPI** | `/simple/<pkg>/`, `/packages/<ver>/` | `/pypi/`, `/simple/` (глобально) | `~/.cache/pip` |
| **Debian** | `/pool/<comp>/`, `/pool/<pkg>` | `/dists/`, `/pool/` (глобально) | `/var/lib/apt/lists/` |
| **RPM** | `/packages/<arch>/<pkg>`, `/packages/debug/` | `/repodata/`, `/packages/` (глобально) | `/var/cache/yum` |

---
Если вам нужно:
- Настроить **зеркалирование с фильтрацией по semver** (например, разрешить только `>=2.0.0 <3.0.0`)
- Интегрировать правила с **GitLab CI / Jenkins** (автогенерация правил из `.allowed-packages`)
- Настроить **Nginx/Traefik** для rate-limiting + IP-geo блокировки перед Nexus

Уточните задачу, и я подготовлю точный конфиг под ваш стек.