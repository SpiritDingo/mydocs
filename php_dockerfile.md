Вот конфигурационные файлы для развертывания PHP-окружения с Apache, PHP-FPM и Cron:

Dockerfile

```dockerfile
FROM php:8.2-apache

# Установка системных зависимостей
RUN apt-get update && apt-get install -y \
    cron \
    libbz2-dev \
    libcurl4-openssl-dev \
    libfreetype6-dev \
    libicu-dev \
    libjpeg62-turbo-dev \
    libldap2-dev \
    libonig-dev \
    libpng-dev \
    libpq-dev \
    libssh2-1-dev \
    libsqlite3-dev \
    libssl-dev \
    libwebp-dev \
    libxml2-dev \
    libxslt1-dev \
    libzip-dev \
    unixodbc-dev \
    unzip \
    wget \
    gnupg \
    && rm -rf /var/lib/apt/lists/*

# Установка расширений PHP
RUN docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp \
    && docker-php-ext-configure ldap --with-libdir=lib/x86_64-linux-gnu/ \
    && docker-php-ext-install -j$(nproc) \
    bcmath \
    bz2 \
    calendar \
    ctype \
    curl \
    dom \
    exif \
    fileinfo \
    ftp \
    gd \
    gettext \
    iconv \
    intl \
    ldap \
    mbstring \
    mysqli \
    opcache \
    pcntl \
    pdo \
    pdo_mysql \
    pdo_pgsql \
    pdo_sqlite \
    posix \
    readline \
    shmop \
    simplexml \
    soap \
    sockets \
    sysvmsg \
    sysvsem \
    sysvshm \
    xml \
    xmlreader \
    xmlwriter \
    xsl \
    zip

# Установка SQL Server драйверов (pdo_sqlsrv, sqlsrv)
RUN curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - \
    && curl https://packages.microsoft.com/config/debian/11/prod.list > /etc/apt/sources.list.d/mssql-release.list \
    && apt-get update \
    && ACCEPT_EULA=Y apt-get install -y msodbcsql18 mssql-tools18 unixodbc-dev \
    && pecl install sqlsrv pdo_sqlsrv \
    && docker-php-ext-enable sqlsrv pdo_sqlsrv

# Установка SSH2 расширения
RUN pecl install ssh2-1.3.1 \
    && docker-php-ext-enable ssh2

# Установка Redis (опционально)
# RUN pecl install redis \
#     && docker-php-ext-enable redis

# Настройка Apache для работы с PHP-FPM
RUN a2enmod proxy_fcgi setenvif rewrite \
    && a2enconf php8.2-fpm

# Настройка PHP
COPY php.ini /usr/local/etc/php/conf.d/custom.ini

# Настройка cron
RUN mkdir -p /var/log/cron \
    && touch /var/log/cron/cron.log

# Создание директории для приложения
WORKDIR /var/www/html

# Копирование скриптов и настройка прав
COPY . /var/www/html/
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html

# Копирование cron файла
COPY cronjobs /etc/cron.d/cronjobs
RUN chmod 0644 /etc/cron.d/cronjobs \
    && crontab /etc/cron.d/cronjobs

# Запуск cron в фоне
RUN echo '#!/bin/bash\n\
cron\n\
apache2-foreground\n\
' > /usr/local/bin/docker-entrypoint.sh \
    && chmod +x /usr/local/bin/docker-entrypoint.sh

ENTRYPOINT ["docker-entrypoint.sh"]
```

docker-compose.yml

```yaml
version: '3.8'

services:
  php-apache:
    build: .
    container_name: php-apache-app
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./src:/var/www/html
      - ./logs/apache2:/var/log/apache2
      - ./logs/php:/var/log/php
      - ./php.ini:/usr/local/etc/php/conf.d/custom.ini
      - ./cronjobs:/etc/cron.d/cronjobs
      - ./ssl:/etc/ssl/certs
    environment:
      - APACHE_RUN_USER=www-data
      - APACHE_RUN_GROUP=www-data
      - TZ=Europe/Moscow
    restart: unless-stopped
    networks:
      - app-network

  # Пример с базой данных MySQL
  db:
    image: mysql:8.0
    container_name: mysql-db
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: app_db
      MYSQL_USER: app_user
      MYSQL_PASSWORD: userpassword
    volumes:
      - mysql-data:/var/lib/mysql
      - ./mysql-init:/docker-entrypoint-initdb.d
    ports:
      - "3306:3306"
    networks:
      - app-network
    restart: unless-stopped

  # Пример с phpMyAdmin
  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    container_name: phpmyadmin
    environment:
      PMA_HOST: db
      PMA_PORT: 3306
      UPLOAD_LIMIT: 128M
    ports:
      - "8080:80"
    networks:
      - app-network
    restart: unless-stopped
    depends_on:
      - db

networks:
  app-network:
    driver: bridge

volumes:
  mysql-data:
    driver: local
```

Дополнительные файлы конфигурации:

php.ini (базовая конфигурация)

```ini
; Настройки отображения ошибок
display_errors = On
display_startup_errors = On
error_reporting = E_ALL
log_errors = On

; Настройки памяти
memory_limit = 256M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 300
max_input_time = 300

; Настройки сессий
session.save_handler = files
session.save_path = "/tmp"
session.gc_maxlifetime = 1440

; Настройки OPCache
opcache.enable=1
opcache.memory_consumption=128
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=10000
opcache.revalidate_freq=2
opcache.fast_shutdown=1

; Дополнительные настройки
date.timezone = Europe/Moscow
```

cronjobs (файл задач cron)

```
# Пример задачи cron - выполнение каждые 5 минут
*/5 * * * * www-data /usr/local/bin/php /var/www/html/scripts/cron-task.php >> /var/log/cron/cron.log 2>&1

# Очистка логов каждую неделю
0 0 * * 0 www-data find /var/log/apache2/ -name "*.log" -mtime +7 -delete

# Запуск каждую минуту (пример)
* * * * * www-data /usr/local/bin/php /var/www/html/scripts/minutely-task.php
```

Структура проекта:

```
project/
├── Dockerfile
├── docker-compose.yml
├── php.ini
├── cronjobs
├── src/
│   ├── index.php
│   └── scripts/
│       ├── cron-task.php
│       └── minutely-task.php
├── logs/
│   ├── apache2/
│   └── php/
└── ssl/
```

Команды для запуска:

1. Сборка и запуск:

```bash
docker-compose up -d --build
```

1. Просмотр логов:

```bash
# Логи PHP/Apache
docker-compose logs -f php-apache

# Логи cron
docker exec -it php-apache-app tail -f /var/log/cron/cron.log
```

1. Проверка установленных модулей PHP:

```bash
docker exec -it php-apache-app php -m
```

1. Остановка:

```bash
docker-compose down
```

Примечания:

1. Для модулей pdo_sqlsrv и sqlsrv требуется драйвер ODBC для SQL Server
2. Модуль ssh2 устанавливается через PECL
3. Cron автоматически запускается вместе с контейнером
4. Apache настроен для работы с PHP-FPM
5. Все необходимые системные зависимости автоматически устанавливаются
6. Для SSL сертификатов создана директория ssl/

При необходимости можно добавить другие модули PHP через docker-php-ext-install или pecl install.