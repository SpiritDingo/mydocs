Чтобы установить пакет sqlsrv для PHP 8.0 на Oracle Linux 9, выполните следующие шаги:

1. Обновите систему

```bash
sudo dnf update -y
```

2. Установите PHP 8.0 и необходимые зависимости

Добавьте репозиторий Remi для PHP 8.0:

```bash
sudo dnf install -y \
    https://rpms.remirepo.net/enterprise/remi-release-9.rpm \
    https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```

Включите репозиторий PHP 8.0:

```bash
sudo dnf module enable php:remi-8.0 -y
```

Установите PHP и зависимости:

```bash
sudo dnf install -y \
    php \
    php-cli \
    php-common \
    php-devel \
    php-pear \
    php-json \
    php-xml \
    php-zip \
    unixODBC-devel
```

3. Установите драйверы Microsoft ODBC

Добавьте репозиторий Microsoft:

```bash
sudo curl -o /etc/yum.repos.d/mssql-release.repo \
    https://packages.microsoft.com/config/rhel/9/prod.repo
```

Установите ODBC-драйвер:

```bash
sudo dnf install -y msodbcsql18
# Для проверки версии: 
odbcinst -q -d -n "ODBC Driver 18 for SQL Server"
```

4. Установите sqlsrv и pdo_sqlsrv

Скачайте и установите модули через pecl:

```bash
sudo pecl install sqlsrv
sudo pecl install pdo_sqlsrv
```

Добавьте расширения в конфигурацию PHP:

```bash
echo "extension=sqlsrv.so" | sudo tee /etc/php.d/20-sqlsrv.ini
echo "extension=pdo_sqlsrv.so" | sudo tee /etc/php.d/20-pdo_sqlsrv.ini
```

5. Проверьте установку

```bash
php -m | grep sqlsrv
```

Вы должны увидеть:

```
pdo_sqlsrv
sqlsrv
```

6. Пример подключения к SQL Server

Создайте тестовый скрипт test_sqlsrv.php:

```php
<?php
$serverName = "your_server";
$connectionOptions = [
    "Database" => "your_database",
    "Uid" => "your_username",
    "PWD" => "your_password"
];

$conn = sqlsrv_connect($serverName, $connectionOptions);
if ($conn === false) {
    die(print_r(sqlsrv_errors(), true));
}
echo "Connected successfully!";
?>
```

Решение возможных проблем

1. Ошибка компиляции: Убедитесь, что установлены все зависимости:
   ```bash
   sudo dnf install -y gcc-c++ php-devel unixODBC-devel
   ```
2. Неправильная версия PHP: Проверьте версию PHP:
   ```bash
   php -v
   ```
3. Обновите pecl:
   ```bash
   sudo pecl channel-update pecl.php.net
   ```
4. Права на папки временных файлов:
   ```bash
   sudo chown -R $(whoami):$(whoami) /tmp
   ```

Готово! Теперь у вас установлены драйверы sqlsrv и pdo_sqlsrv для PHP 8.0 на Oracle Linux 9.