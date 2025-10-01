Файл `daemon.json` — это конфигурационный файл Docker-демона, который позволяет настраивать его поведение без изменения аргументов командной строки. Он обычно находится по пути:

- **Linux**: `/etc/docker/daemon.json`
- **Windows (Docker Desktop)**: не используется напрямую, настройки через GUI или WSL2
- **macOS (Docker Desktop)**: аналогично, через GUI

---

### Пример `daemon.json`

```json
{
  "debug": true,
  "log-level": "info",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "insecure-registries": [
    "my-registry.local:5000"
  ],
  "registry-mirrors": [
    "https://mirror.gcr.io"
  ],
  "data-root": "/mnt/docker-data",
  "iptables": false,
  "live-restore": true,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  },
  "features": {
    "buildkit": true
  }
}
```

---

### Пояснение параметров:

| Параметр | Описание |
|----------|----------|
| `debug` | Включает режим отладки демона |
| `log-level` | Уровень логирования (`debug`, `info`, `warn`, `error`, `fatal`) |
| `log-driver` | Драйвер логирования по умолчанию для контейнеров |
| `log-opts` | Опции для драйвера логов (ограничение размера и количества файлов) |
| `storage-driver` | Драйвер хранения образов и слоёв (рекомендуется `overlay2`) |
| `insecure-registries` | Список приватных реестров без HTTPS |
| `registry-mirrors` | Зеркала Docker Hub для ускорения загрузки образов |
| `data-root` | Путь к директории для хранения данных Docker (по умолчанию `/var/lib/docker`) |
| `iptables` | Отключает управление iptables (осторожно! может нарушить сеть) |
| `live-restore` | Позволяет контейнерам продолжать работать при перезапуске демона |
| `default-ulimits` | Настройка лимитов ресурсов по умолчанию |
| `features.buildkit` | Включает BuildKit для сборки образов |

---

### Как применить изменения:

1. Создайте или отредактируйте файл:
   ```bash
   sudo nano /etc/docker/daemon.json
   ```

2. Проверьте корректность JSON:
   ```bash
   sudo docker info  # покажет ошибки при загрузке конфига
   # или
   python3 -m json.tool /etc/docker/daemon.json
   ```

3. Перезапустите Docker:
   ```bash
   sudo systemctl restart docker
   ```

> ⚠️ **Важно**: Некорректный `daemon.json` может привести к невозможности запуска Docker. Всегда делайте резервную копию перед изменением.

Если нужно — могу помочь составить конфиг под ваш конкретный сценарий (например, для приватного реестра, ограничения памяти, смены директории и т.д.).


В Docker **демон (`dockerd`)** сам по себе **не поддерживает встроенную авторизацию** (вроде логина/пароля для доступа к API), но есть несколько способов реализовать **аутентификацию и безопасный доступ** к Docker API. Ниже приведены два основных подхода:

---

## 🔐 1. **Docker Registry с авторизацией** (часто путают с демоном)

Если вы имеете в виду **авторизацию при работе с приватным Docker Registry**, то вот пример настройки `daemon.json` для работы с **защищённым реестром**:

### ✅ Пример `daemon.json` для приватного registry с HTTPS и доверенным сертификатом:

```json
{
  "insecure-registries": [],
  "registry-mirrors": [],
  "tls": true,
  "tlscacert": "/etc/docker/certs.d/my-registry.local/ca.crt",
  "tlscert": "/etc/docker/certs.d/my-registry.local/client.cert",
  "tlskey": "/etc/docker/certs.d/my-registry.local/client.key"
}
```

> ⚠️ Это **не авторизация по логину/паролю**, а **TLS-аутентификация** (mTLS). Для логина/пароля используется команда:
> ```bash
> docker login my-registry.local:5000
> ```
> Учётные данные сохраняются в `~/.docker/config.json`.

---

## 🔒 2. **Защита Docker API (демона) с авторизацией**

По умолчанию Docker API доступен только через Unix-сокет (`/var/run/docker.sock`). Если вы открываете его по TCP (например, `tcp://0.0.0.0:2375`), **это крайне небезопасно без защиты**.

### 🔧 Шаг 1: Настройка TLS + TCP в `daemon.json`

```json
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"],
  "tls": true,
  "tlscacert": "/etc/docker/ca.pem",
  "tlscert": "/etc/docker/server-cert.pem",
  "tlskey": "/etc/docker/server-key.pem",
  "tlsverify": true
}
```

> Это включает **шифрование и проверку клиентских сертификатов**, но **не логин/пароль**.

### 🔐 Шаг 2: Добавление прокси с авторизацией (например, через Nginx)

Docker **не поддерживает HTTP Basic Auth или JWT напрямую**, поэтому для добавления логина/пароля нужно поставить **обратный прокси** (Nginx, Traefik и т.п.).

#### Пример Nginx конфига с Basic Auth:

```nginx
server {
    listen 2377 ssl;
    server_name docker-api.example.com;

    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    auth_basic "Docker API";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://unix:/var/run/docker.sock;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Создайте файл `.htpasswd`:
```bash
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

Теперь доступ к API будет по:
```
https://docker-api.example.com:2377
```
с логином и паролем.

> 💡 Клиенты (например, `curl` или CI/CD) должны использовать:
> ```bash
> curl -u admin:password https://docker-api.example.com:2377/v1.41/containers/json
> ```

---

## 🛡️ Альтернатива: Использовать **Docker Socket Proxy** или **Auth Proxy**

- [**Tecnativa/docker-socket-proxy**](https://github.com/Tecnativa/docker-socket-proxy) — ограничивает права, но не добавляет логин.
- [**authelia**, **oauth2-proxy**, **Traefik + middleware**] — для полноценной авторизации.

---

## ❌ Что НЕЛЬЗЯ сделать в `daemon.json`:

- Указать логин/пароль напрямую.
- Включить HTTP Basic Auth без внешнего прокси.
- Ограничить доступ по IP (только через `iptables` или прокси).

---

## ✅ Резюме

| Задача | Решение |
|-------|--------|
| Авторизация в **приватном registry** | `docker login`, `~/.docker/config.json` |
| Защита **Docker API** | TLS + прокси (Nginx/Traefik) с Basic Auth или OAuth |
| Настройка в `daemon.json` | Только TLS, хосты, логи, storage и т.п. |

---

Если вы уточните, **что именно вы хотите защитить**:
- доступ к `docker pull/push` в свой registry?
- удалённый доступ к `docker run` через API?

— я дам точный пример конфигурации.


Отлично! Разберём оба сценария отдельно, так как они решаются **по-разному**.

---

## 🔐 Сценарий 1: **Доступ к `docker pull/push` в свой registry**  
(авторизация при работе с **приватным Docker Registry**)

### ✅ Цель:
Чтобы пользователи **не могли** делать `docker pull` или `docker push` без логина/пароля.

### 🛠️ Решение:
Использовать **Docker Registry v2 с аутентификацией** (например, через `htpasswd` или OAuth).

---

### Шаг 1: Запустите приватный registry с авторизацией

Пример с **HTTP Basic Auth** через `htpasswd`:

```bash
# Создайте файл с пользователями
docker run --entrypoint htpasswd httpd:2 -Bbn myuser mypassword > auth/htpasswd

# Запустите registry
docker run -d \
  --name registry \
  -v "$(pwd)/auth:/auth" \
  -v "$(pwd)/certs:/certs" \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  -e "REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt" \
  -e "REGISTRY_HTTP_TLS_KEY=/certs/domain.key" \
  -p 5000:5000 \
  registry:2
```

> 💡 Используйте **валидный TLS-сертификат** (Let's Encrypt или внутренний CA). Иначе придётся указывать `insecure-registries`.

---

### Шаг 2: Настройка `daemon.json` на клиентских машинах

Если registry использует **доверенный сертификат (HTTPS)** — `daemon.json` **не нужен**.  
Если **самоподписанный сертификат** — добавьте:

```json
{
  "insecure-registries": [],
  "registry-mirrors": []
}
```

> ❌ Никакой логин/пароль в `daemon.json` **не указывается**!

---

### Шаг 3: Авторизация на клиенте

Пользователь должен выполнить:

```bash
docker login my-registry.local:5000
# Вводит: myuser / mypassword
```

Учётные данные сохраняются в `~/.docker/config.json`.

Теперь:
```bash
docker pull my-registry.local:5000/myimage
docker push my-registry.local:5000/myimage
```

✅ Работает с авторизацией.

---

## 🔐 Сценарий 2: **Удалённый доступ к `docker run` через API**  
(выполнение команд вроде `docker run`, `docker ps` через HTTP API)

### ⚠️ Важно:
Это **очень опасно**, если не защитить должным образом. API даёт **полный контроль** над хостом!

---

### ✅ Цель:
Разрешить удалённое управление Docker через API, но **только авторизованным пользователям**.

### 🛠️ Решение:
Docker **не поддерживает логин/пароль напрямую**, поэтому используем **TLS + прокси с Basic Auth**.

---

### Шаг 1: Настройка `daemon.json` — включить TCP + TLS

```json
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"],
  "tls": true,
  "tlsverify": true,
  "tlscacert": "/etc/docker/ca.pem",
  "tlscert": "/etc/docker/server-cert.pem",
  "tlskey": "/etc/docker/server-key.pem"
}
```

> Это обеспечивает **шифрование и проверку клиентских сертификатов**, но **не логин/пароль**.

---

### Шаг 2: (Опционально) Добавить Nginx с Basic Auth поверх сокета

Если вы **не хотите возиться с TLS-сертификатами**, можно проксировать **локальный сокет** через Nginx с паролем:

#### `nginx.conf`:
```nginx
server {
    listen 2377 ssl;
    server_name _;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    auth_basic "Docker API";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://unix:/var/run/docker.sock;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Создайте `.htpasswd`:
```bash
htpasswd -c /etc/nginx/.htpasswd admin
```

Теперь API доступен по:  
`https://your-server:2377`

---

### Шаг 3: Использование API с авторизацией

#### Вариант A: Через Nginx (Basic Auth)
```bash
curl -u admin:password https://your-server:2377/v1.43/containers/json
```

#### Вариант B: Через TLS (без пароля, но с сертификатами)
```bash
curl \
  --cacert ca.pem \
  --cert client-cert.pem \
  --key client-key.pem \
  https://your-server:2376/v1.43/containers/json
```

> Для `docker` CLI использовать API напрямую **нельзя** — он работает только с сокетом или TLS-хостом.  
> Но вы можете использовать `DOCKER_HOST`:
> ```bash
> export DOCKER_HOST=tcp://your-server:2376
> export DOCKER_TLS_VERIFY=1
> export DOCKER_CERT_PATH=/path/to/certs
> docker ps
> ```

---

## 📌 Итоговая таблица

| Сценарий | Где настраивается авторизация | Файл `daemon.json` нужен? |
|--------|-------------------------------|---------------------------|
| **`docker pull/push` в свой registry** | В самом **registry** (через `REGISTRY_AUTH`) | Только если registry без HTTPS (`insecure-registries`) |
| **Удалённый `docker run` через API** | Через **Nginx/Traefik** или **TLS-сертификаты** | Да — чтобы включить TCP и TLS |

---

## 🔒 Рекомендации по безопасности

1. **Никогда не открывайте порт 2375 без TLS** — это даёт root-доступ любому в сети.
2. Для registry используйте **HTTPS + аутентификацию**.
3. Для API — лучше использовать **mTLS (сертификаты)**, чем Basic Auth.
4. Ограничьте доступ по IP через `iptables` или фаервол.

---

Если нужно — могу дать готовый `docker-compose.yml` для registry с auth или конфиг Nginx целиком.