# Настройка PHP 8 и Apache 2 на Oracle Linux 9 с использованием `yum`/`dnf`

## 1. Подготовка системы

```bash
sudo dnf update -y
```

## 2. Установка Apache (httpd)

```bash
sudo dnf install httpd -y
```

## 3. Установка PHP 8

Oracle Linux 9 использует `dnf` (аналог `yum`), но команды `yum` также работают:

### Вариант 1: Установка PHP 8.0 из стандартных репозиториев
```bash
sudo dnf install php php-common php-opcache php-cli php-gd php-curl \
               php-mysqlnd php-json php-mbstring php-xml php-zip \
               php-intl php-fpm -y
```

### Вариант 2: Установка более новой версии PHP (8.1, 8.2) из репозитория Remi
```bash
# Добавление репозитория Remi
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-9.rpm -y

# Просмотр доступных версий PHP
sudo dnf module list php

# Включение нужной версии (например, PHP 8.2)
sudo dnf module enable php:remi-8.2 -y

# Установка PHP 8.2 с дополнительными модулями
sudo dnf install php php-common php-opcache php-cli php-gd php-curl \
               php-mysqlnd php-json php-mbstring php-xml php-zip \
               php-intl php-fpm -y
```

## 4. Настройка Apache для работы с PHP

Проверьте, что Apache загружает модуль PHP:
```bash
sudo grep -i 'LoadModule php' /etc/httpd/conf/httpd.conf
```

Если строка отсутствует, добавьте в конфигурацию:
```bash
echo "LoadModule php_module modules/libphp.so" | sudo tee -a /etc/httpd/conf/httpd.conf
```

Добавьте обработчик для PHP-файлов:
```bash
echo "AddHandler php-script .php" | sudo tee -a /etc/httpd/conf/httpd.conf
```

Добавьте index.php в список индексных файлов:
```bash
sudo sed -i 's/DirectoryIndex index.html/DirectoryIndex index.html index.php/' /etc/httpd/conf/httpd.conf
```

## 5. Запуск и настройка служб

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl start php-fpm  # если установлен php-fpm
sudo systemctl enable php-fpm # если установлен php-fpm
```

## 6. Настройка firewall

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

## 7. Проверка установки

Создайте тестовый файл:
```bash
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
```

Откройте в браузере: `http://ваш_сервер/info.php`

## 8. Дополнительные настройки

### Настройка php.ini
```bash
sudo vi /etc/php.ini
```

Рекомендуемые изменения:
```ini
date.timezone = Europe/Moscow
memory_limit = 256M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 300
```

После изменений перезагрузите Apache:
```bash
sudo systemctl restart httpd
```

### Установка дополнительных модулей
```bash
sudo dnf install php-pgsql php-sqlite3 php-bcmath php-soap php-pecl-mcrypt -y
```

Теперь у вас должен быть полностью настроенный веб-сервер с PHP 8 на Oracle Linux 9.



Ошибка возникает из-за конфликта версий PHP между установленными пакетами. Вот несколько способов решения:

Способ 1: Полное переключение на PHP 8.1 (рекомендуется)

```bash
# Удаляем конфликтующие пакеты PHP 8.0
sudo dnf remove php-common-8.0.* php-pecl-ssh2

# Устанавливаем PHP 8.1 и необходимые расширения
sudo dnf install php php-common php-pecl-zip php-pecl-ssh2
```

Способ 2: Использование модулей PHP

```bash
# Смотрим доступные модули PHP
dnf module list php

# Переключаемся на PHP 8.1
sudo dnf module reset php
sudo dnf module enable php:8.1
sudo dnf update php-common php-pecl-zip php-pecl-ssh2
```

Способ 3: Принудительное обновление с заменой пакетов

```bash
sudo dnf update --allowerasing
```

Способ 4: Если нужно сохранить PHP 8.0

```bash
# Удаляем проблемные пакеты
sudo dnf remove php-pecl-ssh2 php-pecl-zip

# Переустанавливаем PHP 8.0 и расширения
sudo dnf install php php-common php-pecl-zip php-pecl-ssh2
```

Способ 5: Очистка и переустановка

```bash
# Удаляем все PHP пакеты
sudo dnf remove php*

# Очищаем кеш
sudo dnf clean all

# Устанавливаем PHP заново
sudo dnf install php php-common php-pecl-zip php-pecl-ssh2
```

Проверка после решения:

```bash
# Проверяем версию PHP
php -v

# Проверяем установленные модули
php -m | grep zip
php -m | grep ssh2
```

Рекомендую использовать Способ 1 или 2 - они наиболее безопасны и правильно разрешают конфликт версий. PHP 8.1 обычно предпочтительнее, так как это более новая версия с улучшенной производительностью и безопасностью.