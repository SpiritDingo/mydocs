Вот подробная шпаргалка по использованию curl в Linux. Она объединяет основные команды и полезные примеры для ежедневной работы.

📝 Основные опции curl

Перед использованием примеров полезно ознакомиться с ключевыми опциями. Полный список можно получить командой curl --help.

Опция Описание
-o <file> Сохранить вывод в файл с указанным именем 
-O Сохранить файл под оригинальным именем с сервера 
-I или --head Получить только HTTP-заголовки ответа (без тела) 
-L Следовать перенаправлениям (редиректам) 
-v Подробный вывод (verbose) для отладки 
-s "Тихий" режим (не показывать индикатор прогресса и ошибки) 
-A <string> Указать свой User-Agent 
-H "<header>" Добавить произвольный заголовок в запрос (например, "Content-Type: application/json") 
-d <data> Отправить данные методом POST (формат x-www-form-urlencoded) 
-F <data> Отправить данные как multipart/form-data (эмуляция формы) 
-X <method> Указать HTTP-метод (GET, POST, PUT, DELETE и т.д.) 
-b <file> Отправить Cookie на сервер из файла 
-c <file> Сохранить полученные от сервера Cookie в файл 
-u user:pass Использовать аутентификацию (HTTP Basic, FTP) 
-k или --insecure Игнорировать проверку SSL-сертификатов (для тестов) 
-x <proxy> Использовать прокси-сервер 
--limit-rate <speed> Ограничить скорость загрузки/отправки (например, 100K) 
-C - Продолжить прерванную загрузку с места разрыва 
-w "<format>" Показать дополнительную информацию после завершения (например, %{time_total}) 

💡 Практические примеры использования

1. Скачивание файлов

```bash
# Скачать файл и сохранить под оригинальным именем
curl -O https://example.com/file.zip

# Скачать файл и сохранить под новым именем
curl -o myfile.zip https://example.com/file.zip

# Продолжить прерванную загрузку
curl -C - -O https://example.com/large-file.iso

# Скачать несколько файлов одной командой
curl -O https://example.com/file1.txt -O https://example.com/file2.txt

# Ограничить скорость загрузки (например, 50 КБ/с)
curl --limit-rate 50K -O https://example.com/file.iso
```

Источник: 

2. Тестирование и отладка веб-серверов

```bash
# Получить только заголовки HTTP-ответа
curl -I https://example.com

# Следовать перенаправлениям и получить финальный ответ
curl -L -I https://example.com

# Отправить запрос с пользовательским заголовком
curl -H "User-Agent: MyBot" -H "Accept-Language: en-US" https://example.com

# Вывести подробную информацию о запросе и ответе (для отладки)
curl -v https://example.com

# Проверить доступность порта и получить время отклика
curl -s -o /dev/null -w "HTTP Code: %{http_code}\nTotal Time: %{time_total}s\n" https://example.com
```

Источник: 

3. Работа с API и отправка данных

```bash
# Отправить простой POST-запрос (данные в формате application/x-www-form-urlencoded)
curl -d "param1=value1&param2=value2" https://api.example.com/endpoint

# Отправить POST-запрос с данными в формате JSON
curl -H "Content-Type: application/json" -d '{"param1":"value1", "param2":"value2"}' https://api.example.com/endpoint

# Отправить файл как часть формы (multipart/form-data)
curl -F "file=@/path/to/local/file.txt" https://example.com/upload

# Выполнить запрос с HTTP-аутентификацией
curl -u username:password https://api.example.com/secure

# Использовать другой HTTP-метод (например, PUT или DELETE)
curl -X PUT -d '{"data":"update"}' https://api.example.com/item/123
```

Источник: 

4. Работа с куки (Cookies) и сессиями

```bash
# Сохранить куки, полученные от сервера, в файл
curl -c cookies.txt https://example.com/login

# Отправить запрос с куками из файла (для поддержания сессии)
curl -b cookies.txt https://example.com/dashboard

# Сохранить новые куки и отправить существующие в одном запросе
curl -b cookies.txt -c cookies.txt https://example.com/action
```

Источник: 

5. Доступ к FTP-серверам

```bash
# Скачать файл с FTP-сервера (анонимно или с аутентификацией)
curl -u ftp_user:ftp_password -O ftp://ftp.example.com/file.txt

# Загрузить файл на FTP-сервер
curl -u ftp_user:ftp_password -T local_file.txt ftp://ftp.example.com/
```

Источник: 

🔧 Специфичные сценарии

· Игнорирование SSL-ошибок: Используйте -k или --insecure для тестирования серверов с самоподписанными сертификатами .
· Указание порта: Для доступа к нестандартным портам укажите его в URL: curl http://localhost:8080 .
· Изменение разрешения DNS: Запрос к конкретному IP, как если бы это был другой домен: curl --resolve example.com:80:127.0.0.1 http://example.com .

Надеюсь, эта шпаргалка будет вам полезна. Для получения полной информации обращайтесь к официальной документации (man curl).


Вот примеры сложных сценариев использования cURL с сертификатами и авторизацией, собранные из различных источников.

🔐 Использование клиентских сертификатов

Когда сервер требует для доступа не только логин и пароль, но и клиентский SSL-сертификат, необходимо указать cURL путь к файлам сертификата и ключа.

Пример для PHP: Для аутентификации в API Сбербанка или других защищенных системах код может выглядеть так:

```php
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, "https://secure-resource.example.com/api");
// Указываем путь к сертификату в формате PEM
curl_setopt($ch, CURLOPT_SSLCERT, "/path/to/your/client.crt");
// Указываем путь к закрытому ключу
curl_setopt($ch, CURLOPT_SSLKEY, "/path/to/your/private.key");
// Если ключ защищен паролем, указываем его
curl_setopt($ch, CURLOPT_SSLKEYPASSWD, "your_key_password");
// Рекомендуется проводить полную проверку сертификата сервера
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);
curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 2);
// Указываем доверенный CA-бундл для проверки сертификата сервера
curl_setopt($ch, CURLOPT_CAINFO, "/path/to/cacert.pem");

$response = curl_exec($ch);
if (curl_errno($ch)) {
    echo 'Ошибка: ' . curl_error($ch);
}
curl_close($ch);
```

Конвертация сертификатов: Если у вас есть сертификат и ключ в отдельных файлах (например, sert.crt и sert.key), их можно объединить в один PEM-файл:

```bash
openssl pkcs12 -export -in sert.crt -inkey sert.key -out certificate.p12
openssl pkcs12 -in certificate.p12 -nodes -out combined.pem
```

После этого в коде можно использовать combined.pem с опцией CURLOPT_SSLCERT.

🔑 Базовые методы авторизации

HTTP Basic Auth — простой метод, при котором логин и пароль передаются в заголовке запроса.

· В командной строке: Используется опция -u.
  ```bash
  curl -u "username:password" https://api.example.com/data
  ```
· В PHP: Настраиваются опции CURLOPT_HTTPAUTH и CURLOPT_USERPWD.
  ```php
  curl_setopt($ch, CURLOPT_HTTPAUTH, CURLAUTH_BASIC);
  curl_setopt($ch, CURLOPT_USERPWD, "username:password");
  ```

📨 Авторизация через веб-формы (POST-запрос)

Этот метод имитирует отправку HTML-формы входа, что часто требует работы с куками и динамическими параметрами.

1. Инициализация сессии и получение кук: Сначала выполняется GET-запрос на страницу входа, чтобы получить начальные куки и извлечь скрытые поля формы (например, csrf-токен).
   ```php
   // Функция для выполнения запросов с сохранением кук
   function curl_request($url, $post_data = null) {
       $ch = curl_init();
       curl_setopt($ch, CURLOPT_URL, $url);
       curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
       curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);
       curl_setopt($ch, CURLOPT_COOKIEJAR, dirname(__FILE__).'/cookie.txt'); // Сохраняем куки в файл
       curl_setopt($ch, CURLOPT_COOKIEFILE, dirname(__FILE__).'/cookie.txt'); // Отправляем куки из файла
       curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false); // Внимание! Для продакшена лучше проверить сертификат
   
       if ($post_data) {
           curl_setopt($ch, CURLOPT_POST, true);
           curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($post_data));
       }
       $response = curl_exec($ch);
       curl_close($ch);
       return $response;
   }
   
   // Получаем страницу входа и извлекаем CSRF-токен
   $login_page = curl_request('https://example.com/login');
   // ... парсим $login_page для извлечения скрытых полей формы ...
   ```
2. Отправка данных для входа: Затем отправляется POST-запрос с логином, паролем и полученными данными.
   ```php
   $post_data = [
       'login' => 'your_username',
       'password' => 'your_password',
       'csrf_token' => $extracted_csrf_token // Динамическое значение
   ];
   $result = curl_request('https://example.com/login', $post_data);
   ```
3. Проверка успешной авторизации: Проверьте ответ на наличие признаков успешного входа.
   ```php
   if (strpos($result, 'Добро пожаловать') !== false) {
       echo "Авторизация успешна!";
   } else {
       echo "Ошибка авторизации.";
   }
   ```

⚠️ Важные замечания по безопасности

· Проверка SSL-сертификатов: Отключать проверку (CURLOPT_SSL_VERIFYPEER => false) допустимо только для тестирования. В боевом режиме всегда включайте проверку и указывайте путь к актуальному файлу с доверенными корневыми сертификатами (CURLOPT_CAINFO).
· Добавление корневых сертификатов: Если сервер использует сертификаты, подписанные необщедоступными центрами сертификации (например, НУЦ Минцифры в России), их нужно добавить в систему или явно указать в cURL.
· Хранение учетных данных: Никогда не храните пароли и ключи прямо в коде. Используйте защищенные хранилища или переменные окружения.

Надеюсь, эти примеры помогут вам реализовать сложные сценарии авторизации. Если вам нужна более конкретная помощь для определенного API, уточните, с каким сервисом вы работаете.