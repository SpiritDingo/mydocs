# Настройка вывода лога PHP на экран

Для настройки вывода логов PHP на экран можно использовать несколько подходов:

## 1. Настройки в php.ini

```
display_errors = On
display_startup_errors = On
error_reporting = E_ALL
```

Эти настройки включают отображение всех ошибок на экране.

## 2. Настройки в коде PHP

```php
<?php
// Включение вывода ошибок
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);
```

## 3. Вывод логов вручную

Если вам нужно вывести содержимое лог-файла:

```php
<?php
// Чтение и вывод лог-файла
$logFile = '/path/to/your/php_errors.log';
if (file_exists($logFile)) {
    echo '<pre>' . htmlspecialchars(file_get_contents($logFile)) . '</pre>';
} else {
    echo 'Лог-файл не найден';
}
```

## 4. Использование функции error_log() с выводом на экран

```php
<?php
// Перенаправление ошибок в буфер вывода
function customErrorHandler($errno, $errstr, $errfile, $errline) {
    echo "<b>Error:</b> [$errno] $errstr<br>";
    echo "Error on line $errline in $errfile<br>";
    return true;
}
set_error_handler("customErrorHandler");
```

## 5. Для разработки - использование Whoops

Установите библиотеку Whoops для красивого вывода ошибок:

```bash
composer require filp/whoops
```

И настройте:

```php
<?php
require 'vendor/autoload.php';

$whoops = new \Whoops\Run;
$whoops->pushHandler(new \Whoops\Handler\PrettyPageHandler);
$whoops->register();
```

## Важные замечания

1. Не используйте `display_errors = On` в production-среде из соображений безопасности
2. Для production лучше логировать ошибки в файл:
   ```ini
   log_errors = On
   error_log = /path/to/php_errors.log
   ```
3. Для отладки AJAX-запросов используйте `ini_set('html_errors', 0);` для чистого вывода ошибок

Выбор метода зависит от ваших целей - отладка на локальной машине или мониторинг ошибок на боевом сервере.