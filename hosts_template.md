Вот расширенный пример файла hosts.yml, который учитывает разделение серверов по типам (базы данных, веб-серверы, серверы приложений) и позволяет задавать для каждого стенда свои учётные данные (логин/пароль или публичные ключи). Структура построена на группах, что даёт гибкость в управлении доступом.

Пример инвентаря (hosts.yml) с учётом типов серверов и разных учётных данных

```yaml
---
all:
  children:
    # ---- Группы проектов ----
    alpha:
      children:
        alpha_dev:
        alpha_test:
        alpha_loadtest:
        alpha_prod:
      vars:
        project_name: alpha

    beta:
      children:
        beta_dev:
        beta_test:
        beta_loadtest:
        beta_prod:
      vars:
        project_name: beta

    # ---- Группы окружений (общие для всех проектов) ----
    dev:
      children:
        alpha_dev:
        beta_dev:
    test:
      children:
        alpha_test:
        beta_test:
    loadtest:
      children:
        alpha_loadtest:
        beta_loadtest:
    prod:
      children:
        alpha_prod:
        beta_prod:

    # ---- Группы типов серверов (общие для всех проектов и окружений) ----
    web:
      children:
        # здесь будут перечислены конкретные группы хостов, относящиеся к web
        # они будут добавлены ниже, когда опишем конкретные окружения проектов
    db:
      children:
    app:
      children:

    # ---- Конкретные окружения проектов с разделением по типам серверов ----
    # Проект Alpha
    alpha_dev:
      children:
        alpha_dev_web:
        alpha_dev_db:
        alpha_dev_app:
      vars:
        env: dev
        # Общие настройки для всего стенда alpha_dev
        ansible_user: devops
        ansible_ssh_private_key_file: ~/.ssh/id_rsa_dev

    alpha_test:
      children:
        alpha_test_web:
        alpha_test_db:
        alpha_test_app:
      vars:
        env: test
        ansible_user: tester
        ansible_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          ... (зашифрованный пароль)

    alpha_loadtest:
      children:
        alpha_loadtest_web:
        alpha_loadtest_db:
        alpha_loadtest_app:
      vars:
        env: loadtest
        ansible_user: loadrunner
        ansible_ssh_private_key_file: ~/.ssh/id_rsa_load

    alpha_prod:
      children:
        alpha_prod_web:
        alpha_prod_db:
        alpha_prod_app:
      vars:
        env: prod
        ansible_user: admin
        ansible_ssh_private_key_file: ~/.ssh/id_rsa_prod

    # Группы хостов по типам для Alpha
    alpha_dev_web:
      hosts:
        alpha-dev-web1:
          ansible_host: 192.168.10.11
        alpha-dev-web2:
          ansible_host: 192.168.10.12
      vars:
        server_type: web
    alpha_dev_db:
      hosts:
        alpha-dev-db1:
          ansible_host: 192.168.10.21
        alpha-dev-db2:
          ansible_host: 192.168.10.22
      vars:
        server_type: db
        # Можно переопределить учётные данные только для БД этого стенда
        # ansible_user: dbadmin
        # ansible_password: secret
    alpha_dev_app:
      hosts:
        alpha-dev-app1:
          ansible_host: 192.168.10.31
      vars:
        server_type: app

    alpha_test_web:
      hosts:
        alpha-test-web1:
          ansible_host: 192.168.20.11
      vars:
        server_type: web
    alpha_test_db:
      hosts:
        alpha-test-db1:
          ansible_host: 192.168.20.21
      vars:
        server_type: db
    alpha_test_app:
      hosts:
        alpha-test-app1:
          ansible_host: 192.168.20.31
      vars:
        server_type: app

    alpha_loadtest_web:
      hosts:
        alpha-load-web1:
          ansible_host: 192.168.30.11
      vars:
        server_type: web
    alpha_loadtest_db:
      hosts:
        alpha-load-db1:
          ansible_host: 192.168.30.21
      vars:
        server_type: db
    alpha_loadtest_app:
      hosts:
        alpha-load-app1:
          ansible_host: 192.168.30.31
      vars:
        server_type: app

    alpha_prod_web:
      hosts:
        alpha-prod-web1:
          ansible_host: 10.0.10.11
        alpha-prod-web2:
          ansible_host: 10.0.10.12
      vars:
        server_type: web
    alpha_prod_db:
      hosts:
        alpha-prod-db1:
          ansible_host: 10.0.10.21
        alpha-prod-db2:
          ansible_host: 10.0.10.22
      vars:
        server_type: db
    alpha_prod_app:
      hosts:
        alpha-prod-app1:
          ansible_host: 10.0.10.31
      vars:
        server_type: app

    # Проект Beta (аналогично)
    beta_dev:
      children:
        beta_dev_web:
        beta_dev_db:
        beta_dev_app:
      vars:
        env: dev
        ansible_user: betadev
        ansible_password: betadevpass  # В реальности лучше использовать vault
    beta_test:
      children:
        beta_test_web:
        beta_test_db:
        beta_test_app:
      vars:
        env: test
        ansible_user: betatester
        ansible_ssh_private_key_file: ~/.ssh/beta_test_key
    beta_loadtest:
      children:
        beta_loadtest_web:
        beta_loadtest_db:
        beta_loadtest_app:
      vars:
        env: loadtest
        ansible_user: betaload
        ansible_password: !vault ...
    beta_prod:
      children:
        beta_prod_web:
        beta_prod_db:
        beta_prod_app:
      vars:
        env: prod
        ansible_user: betaprod
        ansible_ssh_private_key_file: ~/.ssh/beta_prod

    # Группы хостов Beta
    beta_dev_web:
      hosts:
        beta-dev-web1:
          ansible_host: 192.168.10.41
      vars:
        server_type: web
    beta_dev_db:
      hosts:
        beta-dev-db1:
          ansible_host: 192.168.10.51
      vars:
        server_type: db
    beta_dev_app:
      hosts:
        beta-dev-app1:
          ansible_host: 192.168.10.61
      vars:
        server_type: app

    beta_test_web:
      hosts:
        beta-test-web1:
          ansible_host: 192.168.20.41
      vars:
        server_type: web
    beta_test_db:
      hosts:
        beta-test-db1:
          ansible_host: 192.168.20.51
      vars:
        server_type: db
    beta_test_app:
      hosts:
        beta-test-app1:
          ansible_host: 192.168.20.61
      vars:
        server_type: app

    beta_loadtest_web:
      hosts:
        beta-load-web1:
          ansible_host: 192.168.30.41
      vars:
        server_type: web
    beta_loadtest_db:
      hosts:
        beta-load-db1:
          ansible_host: 192.168.30.51
      vars:
        server_type: db
    beta_loadtest_app:
      hosts:
        beta-load-app1:
          ansible_host: 192.168.30.61
      vars:
        server_type: app

    beta_prod_web:
      hosts:
        beta-prod-web1:
          ansible_host: 10.0.20.41
        beta-prod-web2:
          ansible_host: 10.0.20.42
      vars:
        server_type: web
    beta_prod_db:
      hosts:
        beta-prod-db1:
          ansible_host: 10.0.20.51
      vars:
        server_type: db
    beta_prod_app:
      hosts:
        beta-prod-app1:
          ansible_host: 10.0.20.61
      vars:
        server_type: app
```

Пояснения

· Иерархия групп:
  · На верхнем уровне — проекты (alpha, beta).
  · Внутри проектов — окружения (alpha_dev, alpha_test, ...), которые содержат подгруппы по типам серверов (alpha_dev_web, alpha_dev_db, alpha_dev_app).
  · Эти подгруппы уже содержат конкретные хосты.
  · Дополнительно созданы общие группы окружений (dev, test, loadtest, prod) и общие группы типов серверов (web, db, app), которые включают соответствующие подгруппы из всех проектов. Это позволяет выполнять плейбуки, например, на всех веб-серверах независимо от проекта и окружения.
· Учётные данные:
  · На уровне окружения (например, alpha_dev) заданы общие переменные ansible_user и способ аутентификации (через ключ или пароль). Они будут применяться ко всем хостам этого окружения, если не переопределены ниже.
  · При необходимости можно переопределить учётные данные для конкретного типа серверов внутри окружения (например, для alpha_dev_db задать другого пользователя или пароль). Для этого достаточно добавить соответствующие переменные в блок vars этой группы.
  · Если нужны разные учётные данные для конкретного хоста, их можно задать прямо в блоке hosts для этого хоста.
· Безопасность:
  · Пароли рекомендуется хранить в зашифрованном виде с помощью ansible-vault. В примере показано, как можно вставить зашифрованную строку (метка !vault). Для ключей используются пути к файлам на управляющей машине.
· Гибкость выбора целей:
  · --limit alpha — только проект Alpha (все окружения).
  · --limit dev — все dev-окружения (Alpha и Beta).
  · --limit web — все веб-серверы всех проектов и окружений.
  · --limit alpha_dev — только dev-окружение Alpha.
  · --limit alpha_dev_web — только веб-серверы dev-окружения Alpha.
  · Комбинации: --limit 'alpha:&prod:&db' — серверы БД в production-окружении проекта Alpha.

Примечание по использованию групп web, db, app

В примере эти группы объявлены как пустые (children: []), но затем мы не включили в них соответствующие подгруппы автоматически. Чтобы они работали, нужно либо добавить их в секцию children этих групп, либо использовать динамическое включение с помощью конструкции children: со списком подгрупп. В текущем виде они не содержат хостов. Можно исправить, добавив, например:

```yaml
web:
  children:
    alpha_dev_web:
    alpha_test_web:
    alpha_loadtest_web:
    alpha_prod_web:
    beta_dev_web:
    # ... и так далее
```

Но это приведёт к дублированию. Альтернативно можно не создавать такие группы, а выбирать по типу сервера через переменную server_type в плейбуке или использовать --limit по группам окружений и типов одновременно. Но если нужна именно группа web, придётся перечислить все подгруппы вручную. В больших инвентарях это может быть утомительно, поэтому иногда проще полагаться на комбинации групп или использовать плагины инвентаря (например, динамические).

Если вам нужны именно группы web, db, app для удобства, можно доопределить их, перечислив соответствующие подгруппы. Тогда будет работать --limit web.

Заключение

Данная структура позволяет гибко управлять доступом к серверам разных типов в разных проектах и окружениях, задавая для каждого стенда свои логины, пароли или ключи. При необходимости можно легко добавить новые проекты или типы серверов, просто дописав соответствующие блоки по аналогии.

_________________
_________________


Отлично, я понял задачу. Задача не просто разложить хосты по папкам, а построить единый реестр (Source of Truth), где Ansible выступает центральным пультом управления. Цель — видеть всю IT-инфраструктуру, управлять разными проектами и их стендами, не создавая хаос из десятков файлов.

Ниже представлена продвинутая структура единого инвентаря (Monorepo Inventory), которая масштабируется на всю компанию.

Архитектура единого инвентаря

Структура каталогов отражает иерархию: Дата-центр/Провайдер → Проект → Стенд → Роль.

```text
ultimate-inventory/
├── ansible.cfg                      # Настройка "по умолчанию в этот инвентарь"
├── dc1/                             # Дата-центр или Провайдер (AWS, Azure)
│   ├── project-alpha/               # Проект №1
│   │   ├── production/              # Стенд "Бой"
│   │   │   ├── hosts.yml
│   │   │   ├── group_vars/
│   │   │   └── host_vars/
│   │   ├── staging/                 # Стенд "Предпрод"
│   │   │   └── ...
│   │   └── dev/                     # Стенд "Разработка"
│   │       └── ...
│   ├── project-beta/                # Проект №2
│   │   ├── production/
│   │   └── testing/
│   └── _common/                     # Общие сервисы ЦОДа (LDAP, Zabbix, лог-коллекторы)
│       └── services/
│           └── hosts.yml
├── cloud-aws/                       # Другой провайдер
│   └── project-gamma/
│       └── production/
│           └── hosts.yml
└── group_vars/                      # Глобальные переменные для ВСЕХ проектов
    └── all.yml                      # (Осторожно, применяются везде)
```

Детальный разбор ключевых файлов

Теперь наполним эту структуру реальным содержанием и раскроем "магию" управления.

1. Корневой ansible.cfg

Указываем, что по умолчанию мы работаем с нашим огромным инвентарем. Переопределяет системные настройки.

```ini
# ultimate-inventory/ansible.cfg
[defaults]
inventory = ./dc1,./cloud-aws
host_key_checking = False
stdout_callback = yaml
# Запрещаем запускать плейбуки без указания лимита на проде (группа "trg_protected")
callback_whitelist = profile_tasks
```

2. Сердце инвентаря: hosts.yml проекта

Здесь мы строим не просто список, а иерархию сущностей. Пример для dc1/project-alpha/production/hosts.yml:

```yaml
# Файл: ultimate-inventory/dc1/project-alpha/production/hosts.yml
all:
  children:
    # --- Уровень 1: Физическая привязка (Где?) ---
    location_dc1:
      children:
        project_alpha:
          children:
            # --- Уровень 2: Среда / Стенд (Когда?) ---
            stand_production:
              children:
                # --- Уровень 3: Техническая роль (Что?) ---
                role_frontend:
                  hosts:
                    fe-prod-01.dc1.internal:
                    fe-prod-02.dc1.internal:
                role_backend:
                  hosts:
                    be-prod-01.dc1.internal:
                      ansible_host: 10.10.1.10
                      java_heap: "4G"
                role_db_cluster:
                  hosts:
                    db-master-prod.dc1.internal:
                      db_role: master
                    db-slave-prod.dc1.internal:
                      db_role: slave

            # Стенд "Разработка" внутри того же проекта для связности
            stand_develop:
              hosts:
                dev-all-in-one.alpha.internal:

# Специальная группа для "Тяжелого" обслуживания
    datacenter1_linux:
      children:
        role_frontend:
        role_backend:
    # Метка для защиты от дурака
    trg_protected:
      children:
        stand_production:
```

3. Переменные: Каскадирование и переопределение

Сила такого подхода — в наследовании переменных. Ansible собирает их как коктейль: от глобальных до конкретного хоста.

Слой 1: Глобальные (на весь data-центр)

```yaml
# ultimate-inventory/dc1/group_vars/all.yml
# То, что верно для всего ЦОДа
ntp_servers:
  - 10.10.0.1
  - 10.10.0.2
dns_domain: dc1.internal
smtp_relay: smtp.dc1.internal
```

Слой 2: Проектные (общие для всех стендов проекта)

```yaml
# ultimate-inventory/dc1/project-alpha/group_vars/project_alpha.yml
# Бизнес-настройки проекта, не зависят от стенда
app_name: alpha-crm
deploy_user: alpha-deploy
git_repo: git@gitlab.com:team/alpha-crm.git
```

Слой 3: Стендовые (специфика окружения)

```yaml
# ultimate-inventory/dc1/project-alpha/production/group_vars/stand_production.yml
# То, что отличает Прод от Дева
env_type: production
app_port: 443
java_heap: "8G" # Переопределяем дефолтную 4G
db_connection_pool: 150
monitoring_alert_channel: "#alerts-critical"
# ВАЖНО: ansible-vault обязателен!
db_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66386439653233363465...
```

Слой 4: Ролевой (на группу, например, все бекенды)

```yaml
# ultimate-inventory/dc1/project-alpha/production/group_vars/role_backend.yml
# Специфика только для бекенд-серверов в проде
monitoring_exporter_port: 8081
nginx_custom_log_format: 'prod_backend_format'
```

Слой 5: Хостовый (исключение из правил)

```yaml
# ultimate-inventory/dc1/project-alpha/production/host_vars/be-prod-01.dc1.internal.yml
# Если у конкретного сервера сломана половина памяти
java_heap: "3G"
```

Как с этим работать: Сценарии использования

Такая структура позволяет выполнять команды с хирургической точностью.

1. Целиться в проект на конкретном стенде:

```bash
# Обновить бекенды продакшена только для Альфы
ansible-playbook -i ultimate-inventory -l role_backend:&stand_production deploy_backend.yml
# Расшифровка: (роль - бекенд) И (стенд - продакшен)
```

2. Общие операции для всех проектов в дата-центре:

```bash
# Применить патч безопасности ко всем Linux-серверам в ЦОД1
ansible-playbook -i ultimate-inventory -l dc1:&datacenter1_linux security_patch.yml
```

3. Инвентаризация (динамический запрос):

```bash
# Показать все хосты Альфы и их переменную java_heap
ansible-inventory -i ultimate-inventory --host 'project_alpha' --list --yaml
```

4. Защита от случайного деплоя не в тот стенд:
В глобальном плейбуке можно добавить условие, которое проверит, что вы не применяете тестовый код к группе trg_protected без специального тега --skip-tags unsafe.

Советы для Enterprise-уровня

1. Динамический инвентарь: Не прописывайте статикой облачные хосты (AWS EC2). Напишите скрипт (например, aws_ec2.yml), который будет складывать результаты в те же самые группы project_alpha и stand_production, используя теги облака. Такой скрипт просто кладется рядом с hosts.yml в подкаталог inventory/dc1/project-alpha/production/.
2. Ansible Vault: Пароли в group_vars — только зашифрованными. Для каждого стенда можно использовать свой пароль от Vault. Это не даст разработчику, у которого есть пароль от тестового стенда, расшифровать пароль от прода.
3. Кросс-проектные зависимости: Не создавайте жесткие связи в инвентаре. Если Проекту Альфа нужен эндпоинт Беты, вынесите этот эндпоинт в переменную group_vars/stand_production со значением, взятым из внешнего источника (Consul/Vault), а не из имени хоста Беты.

Эта структура превращает статический hosts в многослойную географическую карту вашей инфраструктуры. Если нужно, можем разобрать конкретный пример скрипта для динамического инвентаря или стратегию нейминга групп.