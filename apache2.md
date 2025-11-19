Отличная задача! Настройка нескольких виртуальных хостов на одном сервере Apache с реверс-прокси — распространенная практика. Вот как это реализовать.

🏗️ Базовая структура виртуальных хостов

Создайте отдельные конфигурации для каждого домена/поддомена:

Файл: /etc/apache2/sites-available/site1.conf

```apache
<VirtualHost *:80>
    ServerName domain1.com
    ServerAlias www.domain1.com
    
    # Логи для отладки
    ErrorLog ${APACHE_LOG_DIR}/domain1_error.log
    CustomLog ${APACHE_LOG_DIR}/domain1_access.log combined
    
    # Настройки прокси
    ProxyPreserveHost On
    ProxyPass / http://localhost:8080/
    ProxyPassReverse / http://localhost:8080/
    
    # Дополнительные заголовки для приложения
    RequestHeader set X-Forwarded-Proto "http"
</VirtualHost>
```

Файл: /etc/apache2/sites-available/site2.conf

```apache
<VirtualHost *:80>
    ServerName domain2.com
    ServerAlias www.domain2.com
    
    ErrorLog ${APACHE_LOG_DIR}/domain2_error.log
    CustomLog ${APACHE_LOG_DIR}/domain2_access.log combined
    
    ProxyPreserveHost On
    ProxyPass / http://localhost:8081/
    ProxyPassReverse / http://localhost:8081/
    
    RequestHeader set X-Forwarded-Proto "http"
</VirtualHost>
```

Файл: /etc/apache2/sites-available/site3.conf

```apache
<VirtualHost *:80>
    ServerName api.domain1.com
    
    ErrorLog ${APACHE_LOG_DIR}/api_domain1_error.log
    CustomLog ${APACHE_LOG_DIR}/api_domain1_access.log combined
    
    ProxyPreserveHost On
    ProxyPass / http://localhost:3000/
    ProxyPassReverse / http://localhost:3000/
</VirtualHost>
```

🔧 Активация сайтов

```bash
# Активируем все сайты
sudo a2ensite site1.conf
sudo a2ensite site2.conf
sudo a2ensite site3.conf

# Проверяем конфигурацию
sudo apache2ctl configtest

# Перезагружаем Apache
sudo systemctl reload apache2
```

🌐 Пример с разными типами приложений

```apache
# WordPress сайт
<VirtualHost *:80>
    ServerName blog.domain.com
    
    ProxyPreserveHost On
    ProxyPass / http://localhost:8082/
    ProxyPassReverse / http://localhost:8082/
    
    # Для WordPress важно передавать правильные заголовки
    ProxyPassReverse / http://blog.domain.com/
</VirtualHost>

# Node.js приложение
<VirtualHost *:80>
    ServerName app.domain.com
    
    ProxyPreserveHost On
    ProxyPass / http://localhost:3000/
    ProxyPassReverse / http://localhost:3000/
    
    # Для WebSocket соединений
    ProxyPass /ws/ ws://localhost:3000/ws/
    ProxyPassReverse /ws/ ws://localhost:3000/ws/
</VirtualHost>

# Статический сайт + API
<VirtualHost *:80>
    ServerName static.domain.com
    
    # Статика отдается напрямую Apache
    DocumentRoot /var/www/static
    <Directory /var/www/static>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    
    # API проксируется на бэкенд
    ProxyPass /api/ http://localhost:8083/api/
    ProxyPassReverse /api/ http://localhost:8083/api/
</VirtualHost>
```

🔒 SSL для всех доменов (с Let's Encrypt)

```apache
# Домен 1 с SSL
<VirtualHost *:443>
    ServerName domain1.com
    ServerAlias www.domain1.com
    
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/domain1.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/domain1.com/privkey.pem
    
    ProxyPreserveHost On
    ProxyPass / http://localhost:8080/
    ProxyPassReverse / http://localhost:8080/
    
    RequestHeader set X-Forwarded-Proto "https"
</VirtualHost>

# Домен 2 с SSL
<VirtualHost *:443>
    ServerName domain2.com
    ServerAlias www.domain2.com
    
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/domain2.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/domain2.com/privkey.pem
    
    ProxyPreserveHost On
    ProxyPass / http://localhost:8081/
    ProxyPassReverse / http://localhost:8081/
    
    RequestHeader set X-Forwarded-Proto "https"
</VirtualHost>

# Перенаправление HTTP → HTTPS
<VirtualHost *:80>
    ServerName domain1.com
    ServerAlias www.domain1.com
    Redirect permanent / https://domain1.com/
</VirtualHost>

<VirtualHost *:80>
    ServerName domain2.com
    ServerAlias www.domain2.com
    Redirect permanent / https://domain2.com/
</VirtualHost>
```

🎯 Управление конфигурацией

Просмотр активных сайтов:

```bash
sudo apache2ctl -S
```

Структура каталогов:

```
/etc/apache2/
├── sites-available/    # Доступные конфигурации
│   ├── site1.conf
│   ├── site2.conf
│   └── site3.conf
├── sites-enabled/      # Активированные конфигурации (симлинки)
│   ├── site1.conf -> ../sites-available/site1.conf
│   ├── site2.conf -> ../sites-available/site2.conf
│   └── site3.conf -> ../sites-available/site3.conf
└── apache2.conf        # Главный конфигурационный файл
```

💡 Советы по оптимизации

1. Раздельные логи — каждый домен должен иметь свои файлы логов для упрощения отладки
2. Кэширование — настройте кэширование для статических ресурсов
3. Лимиты — установите разумные лимиты для каждого виртуального хоста
4. Мониторинг — используйте разные Access Log форматы для анализа трафика

Такая конфигурация позволит обслуживать множество доменов на одном сервере, перенаправляя трафик на разные внутренние сервисы.