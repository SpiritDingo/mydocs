Ниже представлен готовый комплект для развертывания `devpi-server` с плагином `devpi-constrained` и проектной фильтрацией.

### Структура проекта
```text
devpi-project/
├── docker-compose.yml
├── Dockerfile
├── config/
│   └── policies.yaml      # Политики фильтрации по проектам
└── data/                  # Том для данных (создается автоматически)
```

### 1. Dockerfile
Создает образ с предустановленным плагином и копирует конфигурацию политик.

```dockerfile
FROM python:3.12-slim

# Версии можно закрепить для воспроизводимости
ARG DEVPI_SERVER_VERSION=7.0.*
ARG DEVPI_CONSTRAINED_VERSION=*

RUN pip install --no-cache-dir \
    "devpi-server==${DEVPI_SERVER_VERSION}" \
    "devpi-web==${DEVPI_SERVER_VERSION}" \
    "devpi-constrained==${DEVPI_CONSTRAINED_VERSION}"

# Директория для данных devpi
ENV DEVPI_SERVER_DIRECTORY=/data
VOLUME ["/data"]

# Копируем политики в образ (будут применены при старте)
COPY config/policies.yaml /etc/devpi/policies.yaml

EXPOSE 3141

# Инициализация + запуск сервера + применение политик
CMD ["sh", "-c", "\
    if [ ! -f /data/.serverversion ]; then \
        echo 'Initializing devpi server...' && \
        devpi-init --serverdir /data; \
    fi && \
    echo 'Starting devpi server...' && \
    devpi-server --host 0.0.0.0 --port 3141 --serverdir /data & \
    SERVER_PID=$! && \
    sleep 5 && \
    echo 'Applying constrained policies...' && \
    devpi use http://localhost:3141 && \
    devpi login root --password '' || true && \
    devpi constrained policy set --file /etc/devpi/policies.yaml || echo 'Policy apply skipped or failed' && \
    wait $SERVER_PID \
"]
```

### 2. docker-compose.yml
```yaml
services:
  devpi:
    build: .
    container_name: devpi-server
    restart: unless-stopped
    ports:
      - "3141:3141"
    volumes:
      # Персистентное хранилище пакетов и БД
      - ./data:/data
      # Монтируем политики как volume для hot-reload без пересборки образа
      - ./config/policies.yaml:/etc/devpi/policies.yaml:ro
    environment:
      # Пароль root (обязательно смените в продакшене!)
      - DEVPI_ROOT_PASSWORD=changeme
    healthcheck:
      test: ["CMD", "devpi", "health", "--url", "http://localhost:3141"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s
```

### 3. config/policies.yaml
Пример проектной фильтрации «Default Deny»:

```yaml
policies:
  # === Общая база (транзитивные зависимости, инструменты) ===
  common-base:
    strict: false
    allow:
      - {name: "pip", versions: "*"}
      - {name: "setuptools", versions: "*"}
      - {name: "wheel", versions: "*"}
      - {name: "typing-extensions", versions: ">=4.0"}

  # === Проект: Backend (prod) ===
  backend-prod:
    inherit: "common-base"
    strict: true
    allow:
      - {name: "fastapi", versions: ">=0.100.0,<1.0"}
      - {name: "uvicorn", versions: ">=0.23.0"}
      - {name: "sqlalchemy", versions: ">=2.0"}
      - {name: "pydantic", versions: ">=2.0,<3.0"}
      - {pattern: "^mycompany-backend-.*", versions: "*"}

  # === Проект: ML Pipeline (prod) ===
  ml-prod:
    inherit: "common-base"
    strict: true
    allow:
      - {name: "torch", versions: ">=2.0,<2.5"}
      - {name: "pandas", versions: ">=2.0"}
      - {name: "numpy", versions: ">=1.24,<2.0"}
      - {pattern: "^mycompany-ml-.*", versions: "*"}

  # === Dev-окружения (не строгие, но с ограничениями) ===
  dev-default:
    inherit: "common-base"
    strict: false
    deny:
      - {name: "log4j-core", versions: "<2.17.0", reason: "CVE-2021-44228"}
      - {pattern: "^malicious-.*", reason: "Blocked by security policy"}
```

### Запуск и использование

```bash
# Сборка и запуск
docker compose up -d --build

# Проверка статуса
docker compose logs -f devpi

# Применение политик после изменения файла (без перезапуска контейнера)
docker compose exec devpi sh -c "\
    devpi use http://localhost:3141 && \
    devpi login root --password changeme && \
    devpi constrained policy set --file /etc/devpi/policies.yaml"

# Привязка политик к индексам (однократно после создания индексов)
docker compose exec devpi sh -c "\
    devpi use http://localhost:3141 && \
    devpi login root --password changeme && \
    devpi index myuser/backend-prod bases=root/pypi && \
    devpi constrained index link myuser/backend-prod --policy backend-prod && \
    devpi index myuser/ml-prod bases=root/pypi && \
    devpi constrained index link myuser/ml-prod --policy ml-prod"
```

### Использование клиентами
```bash
# Установка из проекта backend (только разрешённые пакеты)
pip install -i http://localhost:3141/myuser/backend-prod/+simple/ fastapi

# Попытка установить неразрешённый пакет → ошибка
pip install -i http://localhost:3141/myuser/backend-prod/+simple/ flask
# ERROR: No matching distribution found for flask (blocked by constrained policy)
```

### ⚠️ Важные замечания

| Аспект | Рекомендация |
| :--- | :--- |
| **Пароль root** | В примере используется env-var. Для продакшена используйте Docker Secrets или Vault. |
| **Транзитивные зависимости** | При `strict: true` **все** зависимости должны быть в `allow`. Генерируйте полный список через `pip freeze` или `pip list --format=freeze`. |
| **Hot-reload политик** | Файл смонтирован как volume — изменение на хосте требует только повторного `policy set`, пересборка образа не нужна. |
| **Первое включение** | `devpi-init` выполняется только если `/data` пуст. При повторном запуске данные сохраняются. |
| **Бэкапы** | Регулярно бэкапьте том `./data` — это вся БД пакетов и метаданных. |
| **Логирование блокировок** | Смотрите `docker compose logs devpi \| grep DENIED` для аудита отклонённых запросов. |

> 💡 **Совет:** Перед переводом проекта на `strict: true` запустите симуляцию внутри контейнера:
> ```bash
> docker compose exec devpi sh -c "\
>     devpi use http://localhost:3141/myuser/backend-prod && \
>     devpi constrained simulate myuser/backend-prod -r /path/to/requirements.txt"
> ```
> Это покажет все пакеты, которые будут заблокированы, до реального применения политики.