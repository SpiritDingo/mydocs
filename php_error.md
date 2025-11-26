Чтобы исправить ошибку PHP Warning: include_once(...): Failed to open stream, выполните следующие шаги:

1. Проверьте существование файла

Убедитесь, что файл mysqli.class.php действительно существует в указанной директории ../db/. Используйте абсолютный путь для проверки:

```bash
ls /var/www/html/db/mysqli.class.php
```

Если файла нет, скачайте/создайте его или исправьте путь.

---

2. Исправьте путь к файлу

Используйте абсолютные пути или проверенные относительные пути. Варианты:

• Абсолютный путь через $_SERVER['DOCUMENT_ROOT']:

```php
include_once($_SERVER['DOCUMENT_ROOT'] . '/db/mysqli.class.php');
```

• Проверенный относительный путь:

Если структура проекта такая:

```
/var/www/html/
  ├── db/mysqli.class.php
  └── View/your_script.php
```

Тогда используйте:

```php
include_once(__DIR__ . '/../db/mysqli.class.php');
```

__DIR__ — это абсолютный путь к директории текущего файла.

---

3. Проверьте права доступа

Убедитесь, что PHP имеет права на чтение файла:

```bash
chmod 644 /var/www/html/db/mysqli.class.php
```

---

4. Проверьте автозагрузку классов (альтернативное решение)

Если вы используете композер или современный фреймворк, вместо прямого включения файла используйте автозагрузку:

```php
// Пример с Composer
require_once __DIR__ . '/../vendor/autoload.php';
```

---

5. Обработка ошибок (временное решение)

Чтобы скрыть предупреждение (не рекомендуется для продакшена), можно использовать:

```php
if (file_exists(__DIR__ . '/../db/mysqli.class.php')) {
    include_once(__DIR__ . '/../db/mysqli.class.php');
} else {
    die('Файл mysqli.class.php не найден!');
}
```

---

Пример итогового кода

```php
// Исправленный вариант с абсолютным путем
$db_file = $_SERVER['DOCUMENT_ROOT'] . '/db/mysqli.class.php';
if (file_exists($db_file)) {
    include_once($db_file);
} else {
    error_log("Файл не найден: " . $db_file);
    // Действия при ошибке
}
```

Если проблема не решена, проверьте:

· Корректность имени файла (регистр букв в Linux важен).
· Настройки open_basedir в php.ini.
· Логи веб-сервера для деталей (/var/log/apache2/error.log или /var/log/nginx/error.log).



Ошибка указывает, что Apache не может подключиться к PHP-FPM через сокет /run/php-fpm/www.sock. Вот пошаговое решение:

1. Проверьте статус PHP-FPM

Убедитесь, что служба php-fpm запущена:

```bash
systemctl status php-fpm
```

Если она не активна, запустите её:

```bash
systemctl start php-fpm
```

2. Проверьте путь к сокету

Убедитесь, что сокет существует по указанному пути:

```bash
ls -l /run/php-fpm/www.sock
```

Если файл отсутствует, проверьте конфигурацию PHP-FPM.

3. Настройки PHP-FPM

Откройте конфигурационный файл PHP-FPM (путь зависит от ОС):

```bash
nano /etc/php/7.x/fpm/pool.d/www.conf  # Для Ubuntu/Debian
nano /etc/php-fpm.d/www.conf           # Для CentOS/RHEL
```

Найдите параметр listen и убедитесь, что он соответствует пути из ошибки:

```ini
listen = /run/php-fpm/www.sock
```

4. Права доступа к сокету

Убедитесь, что сокет доступен для пользователя/группы, под которыми работает Apache:

```bash
ls -l /run/php-fpm/
```

Пример правильных прав:

```bash
srw-rw---- 1 www-data www-data 0 Nov 26 18:10 /run/php-fpm/www.sock
```

Если права неправильные, в файле www.conf укажите:

```ini
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
```

5. Перезапустите службы

После изменений перезапустите PHP-FPM и Apache:

```bash
systemctl restart php-fpm
systemctl restart apache2   # Для Ubuntu/Debian
systemctl restart httpd     # Для CentOS/RHEL
```

6. Проверьте конфигурацию Apache

Убедитесь, что в виртуальном хосте Apache указан правильный путь к сокету:

```apache
ProxyPassMatch "^/(.*\.php)$" "unix:/run/php-fpm/www.sock|fcgi://localhost/var/www/html"
```

7. SELinux (для CentOS/RHEL)

Если SELinux включён, проверьте контексты файлов:

```bash
ls -Z /run/php-fpm/www.sock
```

Разрешите доступ через SELinux:

```bash
setsebool -P httpd_execmem on
```

8. Проверьте логи

Если проблема остаётся, изучите логи:

```bash
tail -f /var/log/php-fpm/error.log  # Логи PHP-FPM
tail -f /var/log/httpd/error_log    # Логи Apache (CentOS)
tail -f /var/log/apache2/error.log  # Логи Apache (Ubuntu)
```

9. Альтернативное решение

Если проблема не решена, настройте PHP-FPM на использование TCP-порта вместо сокета:

· В www.conf измените:
  ```ini
  listen = 127.0.0.1:9000
  ```
· В Apache укажите:
  ```apache
  ProxyPassMatch "^/(.*\.php)$" "fcgi://127.0.0.1:9000/var/www/html"
  ```

После всех изменений перезапустите службы и проверьте работу Nextcloud.




Проблема в том, что директория /run/php-fpm не существует, что означает, что PHP-FPM либо не установлен, либо не запускается правильно. Вот решение:

1. Проверьте установку PHP-FPM

```bash
# Для Ubuntu/Debian
dpkg -l | grep php-fpm

# Для CentOS/RHEL
rpm -qa | grep php-fpm
```

Если PHP-FPM не установлен, установите его:

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install php-fpm

# CentOS/RHEL
sudo yum install php-fpm
# или для CentOS 8+/RHEL 8+
sudo dnf install php-fpm
```

2. Запустите PHP-FPM

```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
```

3. Проверьте статус службы

```bash
sudo systemctl status php-fpm
```

Если служба не запускается, проверьте логи:

```bash
sudo journalctl -u php-fpm -f
```

4. Проверьте создание сокета

После запуска PHP-FPM проверьте:

```bash
sudo ls -la /run/php/
```

На некоторых системах сокет может находиться в /run/php/ вместо /run/php-fpm/.

5. Настройте правильный путь в Apache

Если сокет находится в другом месте, обновите конфигурацию Apache.

Найдите правильный путь к сокету:

```bash
sudo find / -name "*.sock" -type s 2>/dev/null | grep php
```

Или проверьте конфигурацию PHP-FPM:

```bash
sudo grep -r "listen" /etc/php/*/fpm/pool.d/
```

6. Обновите конфигурацию Apache

В конфигурации Nextcloud в Apache измените путь к сокету:

```apache
# Вместо
ProxyPassMatch "^/(.*\.php)$" "unix:/run/php-fpm/www.sock|fcgi://localhost/var/www/html"

# Используйте правильный путь, например:
ProxyPassMatch "^/(.*\.php)$" "unix:/run/php/php8.1-fpm.sock|fcgi://localhost/var/www/html"
```

7. Альтернативное решение - использование TCP

Если проблемы со сокетом продолжаются, настройте PHP-FPM на использование TCP:

В файле пула PHP-FPM (/etc/php/*/fpm/pool.d/www.conf):

```ini
listen = 127.0.0.1:9000
```

В Apache:

```apache
ProxyPassMatch "^/(.*\.php)$" "fcgi://127.0.0.1:9000/var/www/html"
```

8. Перезапустите службы

```bash
sudo systemctl restart php-fpm
sudo systemctl restart apache2   # Ubuntu/Debian
sudo systemctl restart httpd     # CentOS/RHEL
```

9. Проверьте работу

```bash
# Проверьте, что PHP-FPM слушает
sudo ss -tlnp | grep php
# или
sudo netstat -tlnp | grep php
```

После выполнения этих шагов Nextcloud должен работать корректно. Если проблема сохраняется, проверьте версию PHP и убедитесь, что все пакеты совместимы.



Вот подробная инструкция по настройке Apache для Nextcloud с PHP-FPM:

1. Установка необходимых компонентов

Для Ubuntu/Debian:

```bash
sudo apt update
sudo apt install apache2 php-fpm php-curl php-cli php-mysqlnd \
php-gd php-json php-curl php-zip php-xml php-mbstring php-bz2 \
php-intl php-gmp php-bcmath php-imagick libapache2-mod-php
```

Для CentOS/RHEL:

```bash
sudo yum install httpd php-fpm php-curl php-cli php-mysqlnd \
php-gd php-json php-common php-zip php-xml php-mbstring \
php-bz2 php-intl php-gmp php-bcmath php-pecl-imagick
```

2. Включение необходимых модулей Apache

```bash
# Ubuntu/Debian
sudo a2enmod proxy_fcgi setenvif rewrite headers env dir mime
sudo a2enconf php8.1-fpm  # Замените на вашу версию PHP

# CentOS/RHEL
sudo systemctl enable --now httpd php-fpm
```

3. Настройка PHP-FPM

Проверьте конфигурацию PHP-FPM:

```bash
sudo nano /etc/php/8.1/fpm/pool.d/www.conf  # Ubuntu/Debian
sudo nano /etc/php-fpm.d/www.conf           # CentOS/RHEL
```

Убедитесь, что есть следующие настройки:

```ini
; Использование сокета
listen = /run/php/php8.1-fpm.sock

; Или использование TCP
; listen = 127.0.0.1:9000

listen.owner = www-data
listen.group = www-data
listen.mode = 0660

user = www-data
group = www-data

pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
```

4. Конфигурация виртуального хоста Apache

Создайте файл конфигурации для Nextcloud:

Ubuntu/Debian:

```bash
sudo nano /etc/apache2/sites-available/nextcloud.conf
```

CentOS/RHEL:

```bash
sudo nano /etc/httpd/conf.d/nextcloud.conf
```

Содержимое конфигурационного файла:

```apache
<VirtualHost *:80>
    ServerName your-domain.com
    ServerAdmin admin@your-domain.com
    
    # Основная директория Nextcloud
    DocumentRoot /var/www/nextcloud
    
    # Логи
    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
    
    # Настройки директории
    <Directory /var/www/nextcloud/>
        Options +FollowSymlinks
        AllowOverride All
        
        <IfModule mod_dav.c>
            Dav off
        </IfModule>
        
        SetEnv HOME /var/www/nextcloud
        SetEnv HTTP_HOME /var/www/nextcloud
        
        # Базовые настройки доступа
        Require all granted
    </Directory>
    
    # Обработка PHP через FPM с использованием сокета
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.1-fpm.sock|fcgi://localhost"
    </FilesMatch>
    
    # Альтернатива: использование TCP
    # <FilesMatch \.php$>
    #    SetHandler "proxy:fcgi://127.0.0.1:9000"
    # </FilesMatch>
    
    # Настройки Proxy для FPM
    ProxyPassMatch "^/(.*\.php)$" "unix:/run/php/php8.1-fpm.sock|fcgi://localhost/var/www/nextcloud"
    
    # Безопасность заголовков
    <IfModule mod_headers.c>
        Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
        Header always set Referrer-Policy "no-referrer"
        Header always set X-Content-Type-Options "nosniff"
        Header always set X-Download-Options "noopen"
        Header always set X-Frame-Options "SAMEORIGIN"
        Header always set X-Permitted-Cross-Domain-Policies "none"
        Header always set X-Robots-Tag "none"
        Header always set X-XSS-Protection "1; mode=block"
    </IfModule>
    
    # Перезапись URL
    RewriteEngine On
    <IfModule mod_rewrite.c>
        RewriteRule .* - [env=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
    </IfModule>
</VirtualHost>
```

5. Настройка прав доступа к файлам

```bash
# Установите правильного владельца
sudo chown -R www-data:www-data /var/www/nextcloud

# Установите правильные права
sudo chmod -R 755 /var/www/nextcloud
sudo chmod -R 750 /var/www/nextcloud/config/
sudo chmod -R 750 /var/www/nextcloud/apps/
sudo chmod -R 750 /var/www/nextcloud/data/
```

6. Активация сайта и перезапуск служб

Ubuntu/Debian:

```bash
sudo a2ensite nextcloud.conf
sudo a2dissite 000-default.conf  # Отключить default сайт если нужно
sudo systemctl restart apache2 php8.1-fpm
```

CentOS/RHEL:

```bash
sudo systemctl restart httpd php-fpm
sudo systemctl enable httpd php-fpm
```

7. Дополнительные настройки безопасности

Добавьте в конфигурацию Apache:

```apache
# Запрет доступа к чувствительным файлам
<Directory /var/www/nextcloud/data>
    Require all denied
</Directory>

<Directory /var/www/nextcloud/config>
    Require all denied
</Directory>

<FilesMatch "\.(md|db|sql|sh|log)$">
    Require all denied
</FilesMatch>

# Настройки для больших файлов
LimitRequestBody 0
<IfModule mod_php.c>
    php_value upload_max_filesize 10G
    php_value post_max_size 10G
    php_value memory_limit 512M
    php_value max_execution_time 3600
    php_value max_input_time 3600
</IfModule>
```

8. Проверка конфигурации

```bash
# Проверка синтаксиса Apache
sudo apache2ctl configtest  # Ubuntu/Debian
sudo httpd -t              # CentOS/RHEL

# Проверка статуса служб
sudo systemctl status apache2
sudo systemctl status php8.1-fpm

# Проверка сокета PHP-FPM
ls -la /run/php/php8.1-fpm.sock
```

9. Настройка SSL (опционально)

```bash
# Установка Certbot для Let's Encrypt
sudo apt install certbot python3-certbot-apache  # Ubuntu/Debian
sudo yum install certbot python3-certbot-apache  # CentOS/RHEL

# Получение сертификата
sudo certbot --apache -d your-domain.com
```

После выполнения этих шагов ваш Nextcloud должен быть доступен через Apache с корректной обработкой PHP через PHP-FPM.