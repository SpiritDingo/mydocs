Ниже — примеры настройки Apache с PHP-FPM (через mod_proxy_fcgi) и основные приёмы работы с HTTP-заголовками: передача входящих заголовков в PHP, проброс авторизации, модификация ответа, проксирование и CORS.

---

1. Базовая связка Apache + PHP-FPM

В современных конфигурациях (Apache 2.4) используют SetHandler или ProxyPassMatch. PHP-скрипты выполняет пул FPM, а Apache выступает фронтендом.

Простой виртуальный хост (файл .conf)

```apache
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/html

    # Передаём все .php файлы в PHP-FPM
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"
    </FilesMatch>

    # Или через ProxyPassMatch (альтернативный вариант):
    # ProxyPassMatch ^/(.*\.php)$ unix:/run/php/php8.2-fpm.sock|fcgi://localhost/var/www/html/$1

    <Directory /var/www/html>
        Require all granted
    </Directory>
</VirtualHost>
```

При такой связке Apache автоматически преобразует HTTP-заголовки запроса в переменные окружения FastCGI вида HTTP_*. В PHP они доступны через $_SERVER:

· $_SERVER['HTTP_HOST']
· $_SERVER['HTTP_USER_AGENT']
· $_SERVER['HTTP_X_CUSTOM_HEADER'] и т. д.

---

2. Передача заголовка Authorization (важно!)

По умолчанию Apache не передаёт заголовок Authorization в PHP-FPM по соображениям безопасности. Это ломает любую Basic/Digest/Bearer-авторизацию в PHP.

Решение – включить передачу (Apache 2.4.13+):

```apache
<VirtualHost *:80>
    ServerName api.example.com
    DocumentRoot /var/www/api

    # Передаём заголовок Authorization
    CGIPassAuth On

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"
    </FilesMatch>
</VirtualHost>
```

Для старых версий Apache можно использовать SetEnvIf:

```apache
SetEnvIf Authorization "(.*)" HTTP_AUTHORIZATION=$1
```

Теперь $_SERVER['HTTP_AUTHORIZATION'] в PHP будет заполнен.

---

3. Установка пользовательских заголовков запроса (перед передачей в PHP)

Можно добавить заголовок в Apache до отправки в бэкенд. Например, для передачи реального IP клиента при использовании прокси:

```apache
<VirtualHost *:80>
    ServerName app.example.com

    # Сохраняем IP клиента
    RequestHeader set X-Real-IP "%{REMOTE_ADDR}s"
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"

    # Или добавляем, если уже есть
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s, %{HTTP_X-Forwarded-For}e"

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php-fpm.sock|fcgi://localhost/"
    </FilesMatch>
</VirtualHost>
```

В PHP они будут доступны как $_SERVER['HTTP_X_REAL_IP'] и $_SERVER['HTTP_X_FORWARDED_FOR'].

---

4. Модификация заголовков ответа (Apache)

Ответ от PHP-FPM можно доработать директивами Header в Apache.

Пример: принудительно убрать заголовок X-Powered-By и добавить CORS

```apache
<VirtualHost *:80>
    ServerName public.api.com
    DocumentRoot /var/www/public

    # Убираем заголовок из ответа PHP
    Header always unset X-Powered-By

    # CORS заголовки (можно задать глобально, не трогая PHP)
    Header always set Access-Control-Allow-Origin "*"
    Header always set Access-Control-Allow-Methods "GET, POST, OPTIONS"
    Header always set Access-Control-Allow-Headers "Authorization, Content-Type"

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php-fpm.sock|fcgi://localhost/"
    </FilesMatch>

    # Обработка preflight запросов OPTIONS
    RewriteEngine On
    RewriteCond %{REQUEST_METHOD} OPTIONS
    RewriteRule ^(.*)$ $1 [R=204,L]
</VirtualHost>
```

Обратите внимание на always – это гарантирует, что заголовок будет добавлен даже для ошибочных ответов (401, 500 и т. д.), а не только при 2xx.

Условная установка заголовков:

```apache
# Добавляем Content-Security-Policy только для HTML
<FilesMatch "\.(html|php)$">
    Header always set Content-Security-Policy "default-src 'self'"
</FilesMatch>

# Кэширование статики, но не PHP
<FilesMatch "\.(css|js|png|jpg|gif|ico)$">
    Header set Cache-Control "max-age=604800, public"
</FilesMatch>
```

---

5. Конфигурация PHP-FPM пула для проброса переменных окружения

Иногда нужно явно задать переменные, которые будут видны в PHP. Это делается в файле пула (например, /etc/php/8.2/fpm/pool.d/www.conf):

```ini
; Пробрасываем кастомные значения
env[APP_ENV] = production
env[MY_CUSTOM_HEADER] = $HTTP_X_CUSTOM_HEADER   ; можно ссылаться на заголовки запроса
```

Но будьте осторожны: переменные, переданные через env, переопределяют значения, пришедшие от Apache. Лучше использовать параметр clear_env = no, чтобы сохранить все переменные окружения, установленные веб-сервером, и дополнить их только необходимыми.

```ini
clear_env = no

; Добавляем дополнительные переменные
env[APPLICATION_MODE] = production
```

---

6. Отладка заголовков

Логирование заголовков в Apache:

```apache
# Логируем выбранные заголовки запроса
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" X-Real-IP:\"%{X-Real-IP}i\"" combined_with_realip
CustomLog /var/log/apache2/access.log combined_with_realip

# Логируем заголовки ответа
Header always note X-Request-ID request_id
LogFormat "... %{request_id}n ..." ...
```

Просмотр заголовков в PHP (отладка):

Временно добавьте скрипт:

```php
<?php
header('Content-Type: text/plain');
print_r($_SERVER);
```

Или используйте getallheaders() (доступна, если PHP работает как Apache module, но для FPM потребуется аналог):

```php
<?php
if (function_exists('getallheaders')) {
    print_r(getallheaders());
} else {
    // эмуляция
    $headers = [];
    foreach ($_SERVER as $name => $value) {
        if (substr($name, 0, 5) == 'HTTP_') {
            $headers[str_replace(' ', '-', ucwords(strtolower(str_replace('_', ' ', substr($name, 5)))))] = $value;
        }
    }
    print_r($headers);
}
```

---

7. Типичные проблемы и их решение

· Заголовок Host не тот – в Apache настройте ProxyPreserveHost On (в контексте ProxyPassMatch), но обычно при SetHandler он передаётся правильно.
· Большие заголовки режутся – увеличьте лимит в пуле FPM: request_terminate_timeout и буферы. Для Apache при использовании mod_proxy_fcgi может потребоваться настройка ProxyFCGIBackendType GENERIC или увеличение LimitRequestFieldSize.
· Заголовки, установленные в PHP (header()), не видны в Apache? Они передаются как есть, Apache может их дополнить через Header append. Если PHP устанавливает X-Foo: bar, а в Apache задано Header always set X-Foo "baz", победит Apache, т. к. он добавляет заголовки после обработки ответа. Используйте append, чтобы добавить новое значение.

---

Итог

Настройка Apache + PHP-FPM для работы с заголовками сводится к нескольким моментам:

1. Базовая передача: включите CGIPassAuth On для авторизации.
2. Модификация запроса: RequestHeader set – передаёт IP, токены и т. д.
3. Корректировка ответа: Header always set/unset/append – CORS, безопасность, кэширование.
4. Переменные окружения: настройка пула FPM (clear_env = no, env[...]).
5. Отладка: формат логов и временный скрипт в PHP.

Приведённые примеры конфигураций работают на Apache 2.4 с модулями proxy, proxy_fcgi, headers, rewrite (включаются при необходимости).


Запрос «исключить заголовок PHP auth pw» скорее всего относится к переменной окружения PHP_AUTH_PW, в которую при HTTP-аутентификации попадает пароль пользователя (Basic auth). В связке Apache + PHP-FPM эта переменная по умолчанию не передаётся (в отличие от mod_php), и это правильное поведение с точки зрения безопасности.

Ниже разбираем, как убедиться, что пароль не попадёт в PHP-скрипты, и как управлять этой переменной.

---

1. Стандартное поведение Apache + PHP-FPM

Когда Apache работает как прокси для PHP-FPM через mod_proxy_fcgi, он не устанавливает переменные PHP_AUTH_USER, PHP_AUTH_PW и AUTH_TYPE для бэкенда, даже если включена директива CGIPassAuth On. Директива CGIPassAuth лишь разрешает передачу заголовка Authorization в виде HTTP_AUTHORIZATION, но не заполняет стандартные CGI-переменные авторизации.

Поэтому в типовой конфигурации PHP-FPM у скриптов нет прямого доступа к паролю через $_SERVER['PHP_AUTH_PW'] – это уже исключено.

```php
// В PHP-FPM под Apache этого не будет:
echo $_SERVER['PHP_AUTH_PW'] ?? 'недоступно'; // всегда недоступно
```

---

2. Если вы по ошибке передали пароль вручную

Иногда администраторы пытаются эмулировать поведение mod_php и в явном виде пробрасывают пароль через SetEnv или RequestHeader:

```apache
# НЕ ДЕЛАЙТЕ ТАК – пароль утекает в скрипты
SetEnv PHP_AUTH_PW "%{AUTHENTICATE_PASS}e"
# или
RequestHeader set PHP_AUTH_PW "%{AUTHENTICATE_PASS}e"
```

Если подобное присутствует в конфигурации, уберите эти директивы. Проверьте все SetEnv, RequestHeader, PassEnv, а также настройки пула FPM (env[...]).

В пуле FPM (например, /etc/php/8.2/fpm/pool.d/www.conf) тоже могло быть добавлено:

```ini
; Удалите или закомментируйте подобные строки
env[PHP_AUTH_PW] = $HTTP_AUTHORIZATION   ; некорректно
```

После изменений перезагрузите Apache и PHP-FPM.

---

3. Полный запрет на передачу любых авторизационных данных в PHP

Если вы вообще не планируете использовать HTTP-авторизацию внутри PHP, держите CGIPassAuth выключенным (по умолчанию выключено):

```apache
<VirtualHost *:80>
    # Явно запрещаем передачу заголовка Authorization
    CGIPassAuth Off

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php-fpm.sock|fcgi://localhost/"
    </FilesMatch>
</VirtualHost>
```

Тогда даже заголовок Authorization не попадёт в $_SERVER['HTTP_AUTHORIZATION'], и пароль останется только на уровне веб-сервера (например, в mod_auth_basic).

---

4. Если нужно убрать PHP_AUTH_PW из логов или отладки

Иногда пароль проскальзывает в логах ошибок через phpinfo(), var_dump($_SERVER) или php-error.log. Чтобы этого избежать:

· Отключите вывод $_SERVER в продакшене.
· Настройте php.ini: expose_php = Off и display_errors = Off.
· Фильтруйте чувствительные ключи в коде, если они всё же появляются.

Специальной опции PHP для удаления PHP_AUTH_PW не требуется, так как по умолчанию его нет в FPM-окружении.

---

Краткий итог

· PHP_AUTH_PW в Apache + PHP-FPM не появляется автоматически – это уже и есть его «исключение».
· Не добавляйте пароль вручную через SetEnv / RequestHeader / env[PHP_AUTH_PW].
· Для полной блокировки авторизационных заголовков отключите CGIPassAuth.
· В коде дополнительно исключите отладочный вывод $_SERVER.

Если под «заголовком» вы имели в виду что-то другое (например, HTTP-заголовок ответа X-Powered-By или Auth-Pw), уточните запрос – я дополню конфигурацию.

PHP_AUTH_PW — это не HTTP-заголовок, а переменная окружения CGI, которую Apache (при использовании mod_php) заполняет паролем из Basic-аутентификации. В браузер она не передаётся, но может быть видна внутри PHP-скриптов через $_SERVER['PHP_AUTH_PW'] или вывод phpinfo(). Если вы хотите скрыть её от PHP-приложений, вот способы.

---

1. Скрытие при использовании mod_php (Apache + встроенный PHP)

Добавьте в конфигурацию виртуального хоста или в .htaccess:

```apache
# Удаляет переменные PHP_AUTH_USER и PHP_AUTH_PW перед передачей в PHP
UnsetEnv PHP_AUTH_PW
UnsetEnv PHP_AUTH_USER
UnsetEnv AUTH_TYPE
```

Это работает в Apache 2.4+. После перезагрузки веб-сервера переменные исчезнут из $_SERVER.

Альтернатива через mod_rewrite (если UnsetEnv не поддерживается):

```apache
RewriteEngine On
# Сбрасываем переменные в пустую строку (не идеально, но скрывает значения)
RewriteRule .* - [E=PHP_AUTH_PW:,E=PHP_AUTH_USER:,E=AUTH_TYPE:]
```

При таком подходе $_SERVER['PHP_AUTH_PW'] будет пустой строкой, а не null.

---

2. Переход на PHP-FPM (рекомендовано)

При использовании связки Apache + PHP-FPM переменные PHP_AUTH_PW и PHP_AUTH_USER по умолчанию отсутствуют. Apache не устанавливает их для проксируемых запросов.
Если вы переключите обработку PHP на FPM, проблема исчезнет автоматически. Базовая настройка:

```apache
<FilesMatch \.php$>
    SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"
</FilesMatch>
```

Подробнее — в первом ответе диалога.

---

3. Защита от случайного вывода

Даже если вы скрыли переменную, убедитесь, что отладочная информация не попадает в браузер:

· Запретите phpinfo() на боевом сервере.
· В php.ini установите:
  ```ini
  expose_php = Off
  display_errors = Off
  ```
· Не выводите $_SERVER целиком в логах или ответах.

---

Важное предупреждение

Многие приложения (например, системы мониторинга, API с Basic Auth) используют PHP_AUTH_USER и PHP_AUTH_PW для аутентификации. Скрыв эти переменные, вы сломаете такую функциональность. Применяйте решение только если вы уверены, что аутентификация реализована иначе (например, через токены в заголовке Authorization).

---

Проверка результата

Выполните простой скрипт:

```php
<?php
var_dump(isset($_SERVER['PHP_AUTH_PW'])); // Должно быть false после правок
?>
```

Если возвращается false — пароль скрыт.