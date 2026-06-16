Ниже приведён пример конфигурации Apache с аутентификацией через LDAPS (LDAP по SSL/TLS, порт 636).
Все пути и доменные имена замените на свои.

1. Необходимые модули

Убедитесь, что модули включены (для Debian/Ubuntu):

```bash
sudo a2enmod ldap authnz_ldap ssl
sudo systemctl restart apache2
```

2. Конфигурация защищённого каталога/виртуального хоста

Поместите блок, например, в /etc/apache2/sites-available/example-ssl.conf или внутрь <VirtualHost *:443>.

```apache
# Путь к PEM-файлу с цепочкой доверенных CA (может быть один сертификат УЦ)
LDAPTrustedGlobalCert CA_BASE64 "/etc/ssl/certs/ldap-ca.pem"
# Режим SSL для LDAP
LDAPTrustedMode SSL

# При необходимости отключить проверку сертификата (ТОЛЬКО для отладки!)
# LDAPVerifyServerCert Off

<VirtualHost *:443>
    ServerName secure.example.com

    # Базовая настройка SSL для HTTPS (кратко)
    SSLEngine On
    SSLCertificateFile      /etc/ssl/certs/server.crt
    SSLCertificateKeyFile   /etc/ssl/private/server.key

    DocumentRoot /var/www/secure

    <Directory /var/www/secure>
        # ---- Аутентификация через LDAPS ----
        AuthType Basic
        AuthName "LDAP Authentication"
        AuthBasicProvider ldap

        # URL для поиска пользователя:
        # ldaps://хост:636/база-поиска?атрибут?область?фильтр
        # uid – логин, который пользователь вводит в форму Basic Auth
        AuthLDAPURL "ldaps://ldap.example.com:636/ou=people,dc=example,dc=com?uid?sub?(objectClass=*)"
        
        # Если требуется привязка (bind) от имени специального пользователя для поиска:
        # AuthLDAPBindDN "cn=reader,dc=example,dc=com"
        # AuthLDAPBindPassword "secret"

        # ---- Авторизация (ограничение доступа) ----
        # Вариант 1: любой успешно аутентифицированный пользователь
        Require valid-user

        # Вариант 2: только члены определённой LDAP-группы
        # AuthLDAPGroupAttribute member
        # Require ldap-group cn=allowed-users,ou=groups,dc=example,dc=com

        # Вариант 3: разрешить нескольких пользователей
        # Require ldap-user user1 user2
    </Directory>
</VirtualHost>
```

3. Пояснения ключевых директив

· LDAPTrustedGlobalCert — указывает файл с доверенными корневыми сертификатами (PEM), чтобы Apache мог проверить сертификат LDAP-сервера. Обычно достаточно экспортировать корневой CA вашего LDAP-сервера.
· LDAPTrustedMode SSL — включает SSL-соединение с LDAP-сервером.
· AuthLDAPURL — полный URL поиска. Формат:
    ldaps://host:port/base_dn?attribute?scope?filter
    attribute — атрибут, который сопоставляется с вводимым именем (часто uid, cn, sAMAccountName для AD).
    scope — sub (по всему поддереву) или one.
    filter — дополнительный LDAP-фильтр для поиска пользователя.
· AuthLDAPBindDN / AuthLDAPBindPassword — если анонимный поиск запрещён, укажите учётные данные для bind.
· Require — управляет доступом после успешной аутентификации.

4. Проверка и применение

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Теперь при обращении к сайту будет запрашиваться логин/пароль, которые проверяются через защищённое LDAPS-соединение.


______


Понял — у вас есть логин (Bind DN) и пароль для подключения к LDAPS. Значит, в конфигурации нужно явно прописать учётные данные, от имени которых Apache будет выполнять поиск пользователя. Ниже — готовый фрагмент с этими директивами (подставьте свои значения).

```apache
# Доверенный сертификат УЦ LDAP-сервера
LDAPTrustedGlobalCert CA_BASE64 "/etc/ssl/certs/ldap-ca.pem"
LDAPTrustedMode SSL

<VirtualHost *:443>
    ServerName secure.example.com
    DocumentRoot /var/www/secure

    SSLEngine On
    SSLCertificateFile      /etc/ssl/certs/server.crt
    SSLCertificateKeyFile   /etc/ssl/private/server.key

    <Directory /var/www/secure>
        AuthType Basic
        AuthName "LDAP Auth"
        AuthBasicProvider ldap

        # --- Поисковая база и сопоставление (подставьте свои значения) ---
        AuthLDAPURL "ldaps://ldap.example.com:636/ou=people,dc=example,dc=com?uid?sub?(objectClass=*)"

        # --- Учётные данные для bind (ваши логин и пароль) ---
        AuthLDAPBindDN "cn=reader,dc=example,dc=com"
        AuthLDAPBindPassword "Ваш_Пароль_Здесь"

        # --- Авторизация ---
        Require valid-user
    </Directory>
</VirtualHost>
```

Что важно заменить:

· AuthLDAPURL – хост, порт, Base DN, атрибут для входа (обычно uid, sAMAccountName для AD).
· AuthLDAPBindDN – Distinguished Name учётной записи для подключения (ваш логин).
· AuthLDAPBindPassword – пароль этой учётной записи.

Если пароль содержит спецсимволы, лучше вынести его в отдельный защищённый файл или использовать AuthLDAPBindPassword exec:..., но для начала можно оставить в конфиге.

После правок проверьте и перезагрузите Apache:

```bash
sudo apache2ctl configtest && sudo systemctl reload apache2
```

Нужна ли помощь с привязкой к конкретной группе или фильтром пользователей?

_________

Логин и пароль для подключения к LDAPS прописываются внутри блока <Directory ...> или <Location ...>, где настроена аутентификация.
Вот конкретные строки:

```apache
# Учётные данные для bind (ваш логин и пароль)
AuthLDAPBindDN "cn=reader,dc=example,dc=com"
AuthLDAPBindPassword "Ваш_Пароль_Здесь"
```

Пояснение:

· AuthLDAPBindDN — Distinguished Name (DN) учётной записи, от имени которой Apache подключается к LDAP-серверу для поиска пользователя, входящего в Basic Auth. Это ваш логин.
· AuthLDAPBindPassword — пароль этой учётной записи.

Куда именно вставлять?
Полный пример секции для защищаемой папки /var/www/secure:

```apache
<Directory /var/www/secure>
    AuthType Basic
    AuthName "LDAP Auth"
    AuthBasicProvider ldap

    # URL поиска пользователей
    AuthLDAPURL "ldaps://ldap.example.com:636/ou=people,dc=example,dc=com?uid?sub?(objectClass=*)"

    # *** ВОТ ЗДЕСЬ ЛОГИН И ПАРОЛЬ ***
    AuthLDAPBindDN "cn=binduser,dc=example,dc=com"
    AuthLDAPBindPassword "Secret123"

    Require valid-user
</Directory>
```

Если вы храните эти данные в отдельном файле (рекомендуется для безопасности), пароль можно задать так:

```apache
AuthLDAPBindPassword exec:/etc/apache2/ldap-secret
```

Файл /etc/apache2/ldap-secret должен содержать только пароль (без переводов строки) и иметь права root:www-data 640.

Главное:
Директивы AuthLDAPBindDN и AuthLDAPBindPassword должны находиться внутри того же контекста, где задан AuthLDAPURL и Require, иначе они не будут действовать. Обычно это <Directory>, <Location> или <VirtualHost>.

Проверить, что всё работает, можно после перезагрузки Apache:

```bash
sudo apache2ctl configtest && sudo systemctl reload apache2
```

Если остались вопросы — уточните, какой именно LDAP (OpenLDAP, Active Directory) и будем настраивать дальше.


________


Ниже — пошаговые инструкции для установки и включения перечисленных модулей в наиболее распространённых дистрибутивах Linux. В большинстве случаев они уже есть в стандартной поставке Apache, и требуется только их активировать.

---

Debian / Ubuntu (Apache именуется apache2)

1. Установка Apache (если ещё не установлен)

```bash
sudo apt update
sudo apt install apache2
```

Все нужные модули уже содержатся в пакете apache2-bin. Отдельно ничего доустанавливать не требуется.

2. Включение модулей

```bash
# mod_cgi (для prefork MPM) и mod_cgid (для worker/event MPM)
sudo a2enmod cgi
sudo a2enmod cgid

# mod_proxy и его подмодули
sudo a2enmod proxy
sudo a2enmod proxy_fcgi
sudo a2enmod proxy_scgi
```

Утилита a2enmod создаёт символические ссылки из /etc/apache2/mods-available/ в /etc/apache2/mods-enabled/.

3. Перезапуск Apache

```bash
sudo systemctl restart apache2
```

4. Проверка

```bash
sudo apachectl -M | grep -E "cgi|proxy_fcgi|proxy_scgi"
```

Вы должны увидеть строки, подтверждающие загрузку модулей.

---

RHEL / CentOS / AlmaLinux / Rocky Linux (Apache именуется httpd)

1. Установка Apache (если необходимо)

В RHEL 7 понадобится пакет из стандартных репозиториев; для RHEL 8/9 модули идут в составе httpd.
Для CentOS 7:

```bash
sudo yum install httpd
```

Для RHEL 8/9 и производных:

```bash
sudo dnf install httpd
```

Все перечисленные модули уже скомпилированы и находятся в /etc/httpd/modules/. Остаётся только прописать их загрузку.

2. Включение модулей через файлы конфигурации

Отредактируйте (или создайте) файл /etc/httpd/conf.modules.d/00-proxy.conf и добавьте/раскомментируйте строки:

```apache
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
LoadModule proxy_scgi_module modules/mod_proxy_scgi.so
```

Для mod_cgi и mod_cgid откройте /etc/httpd/conf.modules.d/00-mpm.conf (или создайте отдельный файл, например /etc/httpd/conf.modules.d/01-cgi.conf) и добавьте:

```apache
LoadModule cgi_module modules/mod_cgi.so
LoadModule cgid_module modules/mod_cgid.so
```

Примечание: cgi_module и cgid_module конфликтуют друг с другом. Обычно включают один из них в зависимости от используемого MPM:

· cgi_module — для prefork MPM,
· cgid_module — для worker или event MPM.

Можно прописать оба: Apache загрузит только совместимый с текущим MPM и проигнорирует второй, выдав предупреждение. Чтобы избежать предупреждений, лучше указать только нужный.

3. Перезапуск Apache

```bash
sudo systemctl restart httpd
```

4. Проверка

```bash
sudo httpd -M | grep -E "cgi|proxy_fcgi|proxy_scgi"
```

---

Что делать, если модуль не обнаружен (редкие случаи)

· Debian/Ubuntu: пакет apache2-bin должен быть установлен. Если модуля почему-то нет, переустановите его:
    sudo apt install --reinstall apache2-bin
· RHEL/CentOS 7: модули proxy_fcgi и proxy_scgi присутствуют в штатном httpd-2.4.6. Если отсутствуют, обновите пакет:
    sudo yum reinstall httpd
· RHEL 8/9: модули в составе httpd-core (зависимость от httpd). Переустановка не требуется.

После включения CGIPassAuth On будет работать именно с теми модулями, через которые идёт обработка запроса (чаще всего proxy_fcgi).

Если планируете использовать .htaccess для CGIPassAuth, убедитесь, что в конфигурации виртуального хоста разрешено AllowOverride AuthConfig.