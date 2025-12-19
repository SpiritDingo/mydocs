Настройка Nexus Repository Manager 3 для работы с репозиториями Ubuntu Pro ESM (Extended Security Maintenance) позволяет централизованно управлять обновлениями безопасности и снизить внешний трафик. Вот пошаговая инструкция:

1. Предварительные требования

На стороне Ubuntu Pro:

· Активная подписка Ubuntu Pro (бесплатная для до 5 машин)
· Сгенерированный токен (на ubuntu.com/pro)

На стороне Nexus:

· Nexus Repository Manager 3 (версия 3.x)
· Административные права

2. Получение credentials Ubuntu Pro

1. Зайдите на ubuntu.com/pro
2. Войдите в свой аккаунт
3. Перейдите в "Credentials" → "APT repositories"
4. Скопируйте:
   · Username: ubuntu
   · Password: длинный токен (начинается с eyJ...)

3. Настройка Nexus Repository Manager

Шаг 1: Создание Blob Store

1. Administration → Repository → Blob Stores
2. Create Blob Store
3. Имя: ubuntu-esm-blobstore
4. Save

Шаг 2: Создание прокси-репозиториев

Для каждой версии Ubuntu создаем отдельный репозиторий:

Пример для Ubuntu 20.04 LTS (Focal):

1. Repository → Repositories → Create repository
2. Выберите apt (proxy)
3. Заполните настройки:

Основные настройки:

· Name: ubuntu-focal-esm-proxy
· URL: https://esm.ubuntu.com/
· Blob store: ubuntu-esm-blobstore

Настройки APT:

· Distribution: focal
· Components: main,restricted,universe,multiverse

Настройки аутентификации:

· Authentication type: Username/Password
· Username: ubuntu
· Password: ваш токен Ubuntu Pro

Примеры URL для разных версий:

· Ubuntu 22.04 (Jammy): https://esm.ubuntu.com/
· Ubuntu 18.04 (Bionic): https://esm.ubuntu.com/

Шаг 3: Создание группового репозитория

1. Create repository → apt (group)
2. Name: ubuntu-esm-group
3. Member repositories: добавьте созданные прокси-репозитории
4. Blob store: выберите созданный blob store

4. Настройка клиентов Ubuntu

Вариант 1: Использование скрипта Ubuntu Pro (рекомендуется)

```bash
# На клиентской машине
sudo pro attach YOUR_UBUNTU_PRO_TOKEN
sudo pro enable esm-infra
```

Вариант 2: Ручная настройка sources.list

```bash
# Резервное копирование оригинального файла
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup

# Добавление репозитория Nexus
sudo tee /etc/apt/sources.list.d/nexus-esm.list << EOF
deb [arch=amd64] http://NEXUS_SERVER/repository/ubuntu-esm-group/ focal main restricted universe multiverse
EOF
```

Вариант 3: Использование аутентификации

Если Nexus требует аутентификацию:

```bash
# Создание файла аутентификации
sudo tee /etc/apt/auth.conf.d/nexus.conf << EOF
machine NEXUS_SERVER/repository/ubuntu-esm-group/
login NEXUS_USER
password NEXUS_PASSWORD
EOF

chmod 600 /etc/apt/auth.conf.d/nexus.conf
```

5. Настройка SSL (если нужно)

Если используется самоподписанный сертификат Nexus:

```bash
# На клиенте Ubuntu
sudo apt-get install ca-certificates
sudo cp nexus-certificate.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

6. Проверка работы

На клиенте:

```bash
sudo apt update
sudo apt upgrade
```

В Nexus:

1. Проверьте логи: Support → Log viewer
2. Проверьте кэшированные пакеты в репозиториях

7. Дополнительные настройки

Health Check:

```bash
# На Nexus
curl -u admin:password -X GET "http://nexus:8081/service/rest/v1/status"
```

Очистка политик:

1. Repository → Cleanup Policies
2. Создайте политику для старых пакетов

8. Скрипт автоматизации настройки

Пример скрипта для настройки клиента:

```bash
#!/bin/bash
NEXUS_URL="http://nexus.example.com"
REPO_NAME="ubuntu-esm-group"
UBUNTU_VERSION=$(lsb_release -cs)

# Настройка sources.list
sudo tee /etc/apt/sources.list.d/nexus.list << EOF
deb ${NEXUS_URL}/repository/${REPO_NAME}/ ${UBUNTU_VERSION} main restricted universe multiverse
deb ${NEXUS_URL}/repository/${REPO_NAME}/ ${UBUNTU_VERSION}-updates main restricted universe multiverse
deb ${NEXUS_URL}/repository/${REPO_NAME}/ ${UBUNTU_VERSION}-security main restricted universe multiverse
EOF

# Обновление пакетов
sudo apt update
```

9. Мониторинг и обслуживание

1. Проверка доступности ESM:

```bash
pro status
```

1. Просмотр использования в Nexus:
   · Storage usage
   · Request statistics
2. Регулярные задачи:
   · Очистка старых пакетов
   · Обновление токенов Ubuntu Pro
   · Мониторинг места на диске

Важные замечания:

1. Токены Ubuntu Pro истекают ежегодно - обновляйте их в настройках Nexus
2. ESM требует активной подписки - без нее доступ к репозиториям прекратится
3. Проверяйте совместимость версий Ubuntu с вашими приложениями
4. Тестируйте обновления в тестовой среде перед применением в production

Эта настройка обеспечит централизованное управление обновлениями безопасности Ubuntu через Nexus Repository Manager.