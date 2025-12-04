Добавление PyPI в качестве proxy-репозитория в Sonatype Nexus 3.x:

1. Через веб-интерфейс Nexus:

1. Войдите в Nexus как администратор
2. Перейдите в Repository → Repositories
3. Нажмите Create repository
4. Выберите тип pypi (proxy)
5. Заполните настройки:

```yaml
Name: pypi-proxy (или другое имя)
Remote storage: https://pypi.org
- или альтернативный URL: https://pypi.org/simple/
```

1. Опциональные настройки в секции Proxy:
   · HTTP port: если нужен отдельный порт
   · Authentication: если требуется авторизация
   · Connection: таймауты и ретраи
2. Нажмите Create repository

2. Через REST API Nexus 3:

```bash
curl -X POST 'http://nexus-server:8081/service/rest/v1/repositories/pypi/proxy' \
  -u admin:password \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "pypi-proxy",
    "online": true,
    "storage": {
      "blobStoreName": "default",
      "strictContentTypeValidation": true
    },
    "proxy": {
      "remoteUrl": "https://pypi.org",
      "contentMaxAge": 1440,
      "metadataMaxAge": 1440
    },
    "negativeCache": {
      "enabled": true,
      "timeToLive": 1440
    },
    "httpClient": {
      "blocked": false,
      "autoBlock": true,
      "connection": {
        "retries": 0,
        "userAgentSuffix": "string",
        "timeout": 60,
        "enableCircularRedirects": false,
        "enableCookies": false
      }
    }
  }'
```

3. Настройка pip для использования proxy:

Добавьте в ~/.pip/pip.conf или глобальный конфиг:

```ini
[global]
index-url = http://nexus-server:8081/repository/pypi-proxy/simple/
trusted-host = nexus-server
```

Или используйте напрямую:

```bash
pip install --index-url http://nexus-server:8081/repository/pypi-proxy/simple/ package-name
```

4. Для Nexus 2.x (устаревшее):

1. Перейдите в Views/Repositories → Repositories
2. Add → Proxy Repository
3. Provider: pypi
4. Repository ID: pypi-proxy
5. Repository Policy: Release
6. Remote Storage Location: https://pypi.org/simple/

5. Настройка группы репозиториев (рекомендуется):

Создайте группу pypi-group, включив:

· pypi-proxy (proxy репозиторий)
· pypi-hosted (если нужен для своих пакетов)

Используйте группу в pip:

```ini
index-url = http://nexus-server:8081/repository/pypi-group/simple/
```

Проверка работы:

```bash
# Тест подключения
curl http://nexus-server:8081/repository/pypi-proxy/

# Тест установки через pip
pip search --index http://nexus-server:8081/repository/pypi-proxy/simple/ requests
```

Примечания:

1. Nexus автоматически кэширует пакеты при первом запросе
2. Для приватных репозиториев настройте аутентификацию в Remote
3. Мониторинг доступен в System → Support → Logs
4. Очистка кэша: Repository → Blob Stores → Compact blob store