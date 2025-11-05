Вот полный пример файла daemon.json для подключения Docker к Nexus registry с авторизацией:

```json
{
  "registry-mirrors": [],
  "insecure-registries": [
    "nexus.local:8082",
    "192.168.1.100:8082",
    "your-nexus-domain:8082"
  ],
  "debug": true,
  "experimental": false,
  "log-level": "info",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "max-concurrent-uploads": 10,
  "max-download-attempts": 5,
  "registry-auth": {
    "nexus.local:8082": {
      "auth": "dXNlcm5hbWU6cGFzc3dvcmQ="
    }
  }
}
```

🔧 Подробное описание параметров:

Основные настройки:

· insecure-registries - список registry, которые используют HTTP вместо HTTPS (обязательно для Nexus по умолчанию)
· nexus.local:8082 - замените на hostname и порт вашего Nexus сервера
· 192.168.1.100:8082 - замените на IP-адрес вашего Nexus сервера

Аутентификация:

Для генерации auth token выполните:

```bash
echo -n 'username:password' | base64
# ИЛИ
echo -n 'admin:your-password' | base64
```

Замените dXNlcm5hbWU6cGFzc3dvcmQ= на полученный base64-код.

🔄 Альтернативный способ настройки аутентификации:

1. Через docker login (рекомендуется):

```bash
docker login nexus.local:8082
# Введите логин и пароль при запросе
```

2. Через файл конфигурации Docker:

Файл ~/.docker/config.json будет содержать:

```json
{
  "auths": {
    "nexus.local:8082": {
      "auth": "dXNlcm5hbWU6cGFzc3dvcmQ="
    }
  }
}
```

⚙️ Дополнительные настройки для продакшн среды:

```json
{
  "insecure-registries": [
    "nexus.company.com:8082"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true",
    "overlay2.size=20G"
  ]
}
```

🔒 Настройка HTTPS (рекомендуется для продакшн):

Если используете HTTPS, уберите insecure-registries и добавьте сертификат:

```bash
# Копирование CA сертификата Nexus
sudo mkdir -p /etc/docker/certs.d/nexus.company.com:8082
sudo cp nexus-ca.crt /etc/docker/certs.d/nexus.company.com:8082/ca.crt
```

🚀 Применение конфигурации:

1. Сохраните файл как /etc/docker/daemon.json
2. Перезапустите Docker демон:

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

1. Проверьте подключение:

```bash
docker login nexus.local:8082
docker pull nexus.local:8082/your-image:tag
```

🛠️ Устранение проблем:

Если возникают ошибки:

· Проверьте, что порт registry доступен: telnet nexus.local 8082
· Убедитесь, что Docker Bearer Token Realm включен в Nexus
· Проверьте логи: journalctl -u docker -f

Эта конфигурация обеспечит надежное подключение Docker к вашему локальному Nexus repository с поддержкой авторизации.


Вот дополнительные варианты конфигурации daemon.json для подключения к Nexus:

🏢 Вариант 2: Для корпоративной среды с несколькими registry

```json
{
  "insecure-registries": [
    "nexus.company.local:8082",
    "nexus.company.local:8083",
    "registry.internal:5000"
  ],
  "registry-mirrors": [
    "https://mirror.gcr.io",
    "https://registry.docker-cn.com"
  ],
  "debug": false,
  "log-level": "warn",
  "storage-driver": "overlay2",
  "data-root": "/var/lib/docker",
  "exec-opts": [
    "native.cgroupdriver=systemd"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3",
    "labels": "production"
  },
  "live-restore": true,
  "max-concurrent-downloads": 10,
  "max-concurrent-uploads": 5,
  "default-address-pools": [
    {
      "base": "172.80.0.0/12",
      "size": 24
    }
  ]
}
```

🔐 Вариант 3: С безопасной аутентификацией через отдельный конфиг

/etc/docker/daemon.json:

```json
{
  "insecure-registries": [
    "nexus.local:8082"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "mtu": 1500,
  "ipv6": false,
  "fixed-cidr-v6": "2001:db8:1::/64",
  "experimental": false,
  "metrics-addr": "127.0.0.1:9323",
  "live-restore": true
}
```

Отдельный файл аутентификации /root/.docker/config.json:

```json
{
  "auths": {
    "nexus.local:8082": {
      "auth": "dXNlcm5hbWU6cGFzc3dvcmQ="
    },
    "nexus.local:8083": {
      "auth": "YWRtaW46cGFzc3dvcmQ="
    }
  },
  "HttpHeaders": {
    "User-Agent": "Docker-Client/19.03.12 (linux)"
  },
  "credsStore": "pass"
}
```

🐳 Вариант 4: Для Docker Swarm с TLS

```json
{
  "insecure-registries": [
    "nexus-swarm.local:8082"
  ],
  "tls": true,
  "tlscacert": "/etc/docker/ssl/ca.pem",
  "tlscert": "/etc/docker/ssl/server-cert.pem",
  "tlskey": "/etc/docker/ssl/server-key.pem",
  "tlsverify": true,
  "cluster-store": "consul://127.0.0.1:8500",
  "cluster-advertise": "eth0:2376",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true",
    "overlay2.size=50G"
  ],
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  }
}
```

🚀 Вариант 5: Высокопроизводительная настройка

```json
{
  "insecure-registries": [
    "nexus-perf.local:8082"
  ],
  "registry-mirrors": [
    "https://registry-1.docker.io",
    "https://mirror.gcr.io"
  ],
  "debug": false,
  "log-level": "error",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true",
    "overlay2.size=100G"
  ],
  "data-root": "/opt/docker",
  "bip": "172.26.0.1/16",
  "default-address-pools": [
    {
      "base": "172.80.0.0/12",
      "size": 24
    }
  ],
  "max-concurrent-downloads": 20,
  "max-concurrent-uploads": 10,
  "max-download-attempts": 5,
  "dns": ["8.8.8.8", "8.8.4.4"],
  "dns-opts": ["timeout:2", "attempts:3"],
  "live-restore": true,
  "oom-score-adjust": -500,
  "exec-opts": [
    "native.cgroupdriver=systemd"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "200m",
    "max-file": "5",
    "compress": "true"
  },
  "features": {
    "buildkit": true
  },
  "builder": {
    "gc": {
      "enabled": true,
      "defaultKeepStorage": "20GB"
    }
  }
}
```

🔧 Вариант 6: Минималистичный для разработки

```json
{
  "insecure-registries": ["nexus-dev:8082"],
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

📝 Скрипт для автоматической настройки:

```bash
#!/bin/bash
# setup-docker-nexus.sh

NEXUS_HOST="nexus.local"
NEXUS_PORT="8082"
USERNAME="docker-user"
PASSWORD="your-password"

# Создаем директорию для сертификатов
sudo mkdir -p /etc/docker/certs.d/${NEXUS_HOST}:${NEXUS_PORT}

# Генерируем auth token
AUTH_TOKEN=$(echo -n "${USERNAME}:${PASSWORD}" | base64)

# Создаем daemon.json
sudo tee /etc/docker/daemon.json <<EOF
{
  "insecure-registries": ["${NEXUS_HOST}:${NEXUS_PORT}"],
  "debug": false,
  "log-level": "info",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  }
}
EOF

# Создаем конфиг аутентификации
mkdir -p ~/.docker
cat > ~/.docker/config.json <<EOF
{
  "auths": {
    "${NEXUS_HOST}:${NEXUS_PORT}": {
      "auth": "${AUTH_TOKEN}"
    }
  }
}
EOF

# Перезапускаем Docker
sudo systemctl daemon-reload
sudo systemctl restart docker

echo "Docker настроен для работы с Nexus ${NEXUS_HOST}:${NEXUS_PORT}"
```

🔍 Проверка конфигурации:

После применения любой конфигурации проверьте:

```bash
# Проверить текущую конфигурацию
docker system info

# Проверить подключение к Nexus
curl -X GET http://nexus.local:8082/v2/_catalog

# Тестовый логин
docker login nexus.local:8082

# Проверить образы в registry
docker images
```

⚠️ Важные замечания:

1. Безопасность: Не используйте insecure-registries в продакшн среде
2. Бэкап: Сохраняйте резервные копии конфигурационных файлов
3. Тестирование: Всегда тестируйте конфигурацию в dev-среде
4. Документация: Ведите документацию по изменениям конфигурации

Выберите вариант, который наиболее соответствует вашим требованиям по безопасности, производительности и функциональности.

Когда Nexus использует 443 порт (стандартный HTTPS порт), конфигурация значительно упрощается, так как не требуется настраивать insecure-registries. Вот оптимальные варианты:

🎯 Вариант 1: Базовая конфигурация для HTTPS

```json
{
  "registry-mirrors": [],
  "debug": false,
  "log-level": "info",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  },
  "live-restore": true,
  "max-concurrent-downloads": 10,
  "experimental": false
}
```

🔐 Вариант 2: С настроенными сертификатами

```json
{
  "debug": false,
  "log-level": "warn",
  "storage-driver": "overlay2",
  "data-root": "/var/lib/docker",
  "exec-opts": [
    "native.cgroupdriver=systemd"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "live-restore": true,
  "max-concurrent-downloads": 20,
  "max-concurrent-uploads": 10,
  "tlsverify": true,
  "tlscacert": "/etc/docker/certs/ca.pem",
  "tlscert": "/etc/docker/certs/client-cert.pem",
  "tlskey": "/etc/docker/certs/client-key.pem"
}
```

🚀 Вариант 3: Высокопроизводительная настройка

```json
{
  "registry-mirrors": [
    "https://nexus.company.com"
  ],
  "debug": false,
  "log-level": "error",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true",
    "overlay2.size=50G"
  ],
  "data-root": "/opt/docker",
  "bip": "172.26.0.1/16",
  "max-concurrent-downloads": 20,
  "max-concurrent-uploads": 10,
  "max-download-attempts": 5,
  "dns": ["8.8.8.8", "8.8.4.4"],
  "dns-opts": ["timeout:2", "attempts:3"],
  "live-restore": true,
  "exec-opts": [
    "native.cgroupdriver=systemd"
  ],
  "features": {
    "buildkit": true
  }
}
```

📝 Настройка аутентификации для порта 443:

1. Docker login (рекомендуется):

```bash
docker login nexus.company.com
# Или с явным указанием порта 443
docker login nexus.company.com:443
```

2. Ручная настройка в ~/.docker/config.json:

```json
{
  "auths": {
    "nexus.company.com": {
      "auth": "dXNlcm5hbWU6cGFzc3dvcmQ="
    },
    "nexus.company.com:443": {
      "auth": "dXNlcm5hbWU6cGFzc3dvcmQ="
    }
  },
  "credsStore": "secretservice"
}
```

🔧 Вариант 4: Минималистичный для продакшн

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "live-restore": true
}
```

⚙️ Настройка сертификатов для HTTPS:

Если используется самоподписанный сертификат:

```bash
# Создание директории для сертификатов
sudo mkdir -p /etc/docker/certs.d/nexus.company.com

# Копирование CA сертификата
sudo cp nexus-ca.crt /etc/docker/certs.d/nexus.company.com/ca.crt

# Или для порта 443 явно
sudo mkdir -p /etc/docker/certs.d/nexus.company.com:443
sudo cp nexus-ca.crt /etc/docker/certs.d/nexus.company.com:443/ca.crt
```

🛠️ Автоматический скрипт настройки:

```bash
#!/bin/bash
# setup-docker-nexus-443.sh

NEXUS_HOST="nexus.company.com"
USERNAME="docker-user"
PASSWORD="your-password"

# Создаем daemon.json для HTTPS
sudo tee /etc/docker/daemon.json <<EOF
{
  "debug": false,
  "log-level": "info",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  },
  "live-restore": true
}
EOF

# Логин в registry (создаст запись в ~/.docker/config.json)
docker login $NEXUS_HOST -u $USERNAME -p $PASSWORD

# Перезапускаем Docker
sudo systemctl daemon-reload
sudo systemctl restart docker

echo "Docker настроен для работы с Nexus по HTTPS (порт 443)"
```

🔄 Проверка подключения:

```bash
# Проверить доступность Nexus
curl -I https://nexus.company.com

# Тестовый логин
docker login nexus.company.com

# Проверить образы
curl -u username:password https://nexus.company.com/v2/_catalog

# Проверить конфигурацию Docker
docker system info
```

🎪 Вариант 5: Для Docker Swarm с HTTPS

```json
{
  "tls": true,
  "tlscacert": "/etc/docker/ssl/ca.pem",
  "tlscert": "/etc/docker/ssl/server-cert.pem",
  "tlskey": "/etc/docker/ssl/server-key.pem",
  "tlsverify": true,
  "cluster-store": "consul://127.0.0.1:8500",
  "cluster-advertise": "eth0:2376",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "live-restore": true
}
```

💡 Ключевые преимущества порта 443:

· ✅ Не требуется insecure-registries
· ✅ Стандартный HTTPS порт
· ✅ Работает через корпоративные прокси
· ✅ Более безопасная конфигурация
· ✅ Проще настройка CI/CD

Важно: При использовании порта 443 убедитесь, что:

· DNS имя корректно разрешается
· SSL сертификат валиден (или добавлен в доверенные)
· Брандмауэр разрешает исходящие подключения на 443 порт

Выберите вариант, который лучше всего подходит для вашей среды!