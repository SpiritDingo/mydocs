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


Ошибка cannot find -lltdl указывает на отсутствие библиотеки libtool-ltdl. Вот решения:

Способ 1: Установка недостающих зависимостей

```bash
# Установка необходимых библиотек для компиляции
sudo dnf install libtool-ltdl-devel unixODBC-devel gcc-c++ make
```

Способ 2: Установка полного набора инструментов разработки

```bash
# Установка групп разработки
sudo dnf groupinstall "Development Tools"
sudo dnf install libtool-ltdl-devel unixODBC-devel
```

Способ 3: Поиск и установка конкретного пакета

```bash
# Поиск пакета, предоставляющего библиотеку ltdl
dnf provides "*/libltdl.so*"

# Установка найденного пакета
sudo dnf install libtool-ltdl
```

Способ 4: Для PHP расширений Microsoft SQL Server

Если вы компилируете sqlsrv или pdo_sqlsrv, установите все зависимости:

```bash
sudo dnf install \
    libtool-ltdl-devel \
    unixODBC-devel \
    gcc-c++ \
    make \
    php-devel \
    openssl-devel
```

Способ 5: Проверка установленных библиотек

```bash
# Проверяем наличие библиотек
ldconfig -p | grep ltdl
find /usr -name "libltdl*" 2>/dev/null
```

После установки зависимостей:

```bash
# Повторяем сборку
pecl install sqlsrv
# или
phpize && ./configure && make && sudo make install
```

Дополнительные рекомендации:

1. Очистите кеш сборки:
   ```bash
   make clean
   phpize --clean
   ```
2. Если используете PECL:
   ```bash
   pecl uninstall sqlsrv
   pecl install sqlsrv
   ```
3. Проверьте версии:
   ```bash
   gcc --version
   make --version
   php -v
   ```

Наиболее вероятное решение - выполнить Способ 1 или 2, чтобы установить все необходимые инструменты разработки и библиотеки.