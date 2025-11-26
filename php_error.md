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