Конечно! Передача переменных из GitLab CI/CD в Ansible — это распространённая задача при автоматизации развертывания и управления инфраструктурой. GitLab позволяет хранить переменные проекта или группы, которые затем можно использовать в пайплайне для настройки Ansible.

Ниже приведены основные способы передачи переменных из GitLab в Ansible с примерами.

1. Передача через --extra-vars в команде ansible-playbook

Это самый простой способ — переменные GitLab CI подставляются в команду запуска плейбука.

.gitlab-ci.yml

```yaml
stages:
  - deploy

deploy:
  stage: deploy
  script:
    - ansible-playbook -i inventory/prod playbook.yml --extra-vars "target_host=$TARGET_HOST app_version=$APP_VERSION"
  only:
    - main
```

Здесь $TARGET_HOST и $APP_VERSION — это переменные GitLab CI/CD (определены в настройках проекта или группы). Они будут подставлены в команду и станут extra-vars для Ansible.

playbook.yml (фрагмент)

```yaml
- hosts: "{{ target_host }}"
  vars:
    version: "{{ app_version }}"
  roles:
    - myapp
```

Роль myapp получит переменные target_host и app_version как обычные переменные Ansible.

2. Создание файла переменных в пайплайне и использование vars_files

Иногда удобнее записать переменные в файл (например, YAML или JSON) и передать его в плейбук.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  script:
    - |
      cat > extra_vars.yml << EOF
      target_host: "$TARGET_HOST"
      app_version: "$APP_VERSION"
      database_url: "$DATABASE_URL"
      EOF
    - ansible-playbook -i inventory/prod playbook.yml --extra-vars "@extra_vars.yml"
```

playbook.yml

```yaml
- hosts: all
  vars_files:
    - extra_vars.yml
  roles:
    - myapp
```

В этом случае переменные из файла будут доступны всем хостам и ролям.

3. Использование переменных окружения GitLab CI внутри Ansible

Ansible автоматически наследует переменные окружения, доступные в процессе выполнения. Вы можете использовать lookup-плагин env или прямое обращение к ansible_env.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  variables:
    APP_ENV: production
    DB_PASSWORD: $DB_PASSWORD  # секретная переменная GitLab
  script:
    - ansible-playbook -i inventory/prod playbook.yml
```

playbook.yml

```yaml
- hosts: all
  tasks:
    - name: Используем переменную окружения
      debug:
        msg: "Окружение: {{ lookup('env', 'APP_ENV') }}"
    - name: Используем ansible_env
      debug:
        msg: "Пароль БД: {{ ansible_env.DB_PASSWORD }}"
```

Роль также может обращаться к переменным окружения через lookup('env', 'VAR') или через ansible_env.VAR.

4. Динамическое формирование inventory из переменных GitLab

Если нужно передать список хостов или групп, можно сгенерировать inventory-файл в пайплайне.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  script:
    - |
      echo "[web]" > inventory/prod
      echo "$WEB_HOSTS" | tr ',' '\n' >> inventory/prod
      echo "[db]" >> inventory/prod
      echo "$DB_HOST" >> inventory/prod
    - ansible-playbook -i inventory/prod playbook.yml
```

Переменные WEB_HOSTS и DB_HOST могут содержать IP-адреса или имена хостов.

playbook.yml

```yaml
- hosts: web
  roles:
    - nginx
- hosts: db
  roles:
    - postgresql
```

5. Передача переменных в роли через vars в плейбуке

Вы можете определить переменные на уровне плейбука и передать их в роль. В GitLab CI эти переменные можно задать через --extra-vars.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  script:
    - ansible-playbook -i inventory/prod playbook.yml --extra-vars "nginx_port=8080 ssl_enabled=true"
```

playbook.yml

```yaml
- hosts: web
  roles:
    - role: nginx
      vars:
        nginx_port: "{{ nginx_port }}"
        ssl_enabled: "{{ ssl_enabled }}"
```

Внутри роли nginx эти переменные будут доступны через стандартные имена (nginx_port, ssl_enabled).

6. Использование group_vars и host_vars, сгенерированных из GitLab

Можно заранее в репозитории хранить шаблоны group_vars или host_vars, а в пайплайне подставлять значения из переменных GitLab с помощью envsubst или Ansible template.

Пример с envsubst

group_vars/all.yml.j2 (шаблон)

```yaml
app_version: ${APP_VERSION}
database_url: ${DATABASE_URL}
```

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  script:
    - export APP_VERSION=$APP_VERSION DATABASE_URL=$DATABASE_URL
    - envsubst < group_vars/all.yml.j2 > group_vars/all.yml
    - ansible-playbook -i inventory/prod playbook.yml
```

7. Использование ansible-vault с паролем из GitLab

Если переменные секретные, их можно зашифровать и хранить в репозитории, а пароль для расшифровки передавать через GitLab CI.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  script:
    - echo "$VAULT_PASSWORD" > vault_pass.txt
    - ansible-playbook -i inventory/prod playbook.yml --vault-password-file vault_pass.txt
```

Важные замечания

· Всегда маскируйте секретные переменные в GitLab (Settings → CI/CD → Variables), чтобы они не отображались в логах.
· Для сложных структур данных (словари, списки) используйте файлы YAML/JSON, а не --extra-vars с inline-строками.
· Помните о безопасности: не выводите секретные значения через debug без необходимости.

Эти примеры покрывают большинство сценариев передачи переменных из GitLab в Ansible роли и плейбуки. Выберите подходящий для вашего проекта.

________
________

Конечно! Вот ещё несколько более продвинутых и специфических способов передачи переменных из GitLab CI/CD в Ansible.

8. Использование Ansible Tower / AWX API

Если у вас есть Ansible Tower или AWX, можно запускать шаблоны заданий через API, передавая extra_vars прямо из GitLab CI. Это удобно для централизованного управления и аудита.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  script:
    - |
      curl -X POST https://tower.example.com/api/v2/job_templates/7/launch/ \
        -H "Authorization: Bearer $TOWER_TOKEN" \
        -H "Content-Type: application/json" \
        -d '{
          "extra_vars": {
            "target_host": "'"$TARGET_HOST"'",
            "app_version": "'"$APP_VERSION"'"
          }
        }'
```

В Tower/AWX шаблон задания уже содержит inventory, плейбук и роли, а переменные передаются динамически.

9. Использование set_fact для импорта переменных окружения

В плейбуке можно явно импортировать все переменные окружения (или выбранные) в факты Ansible. Это полезно, когда вы хотите использовать переменные CI внутри ролей без написания lookup каждый раз.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  variables:
    MY_CUSTOM_VAR: "hello"
    DB_HOST: "db.internal"
  script:
    - ansible-playbook -i inventory/prod playbook.yml
```

playbook.yml

```yaml
- hosts: all
  pre_tasks:
    - name: Импорт всех переменных окружения в факты
      ansible.builtin.set_fact:
        env_vars: "{{ ansible_env }}"
    - name: Импорт конкретных переменных
      ansible.builtin.set_fact:
        my_custom_var: "{{ ansible_env.MY_CUSTOM_VAR }}"
        db_host: "{{ ansible_env.DB_HOST }}"
  roles:
    - myapp
```

В роли myapp теперь можно обращаться к my_custom_var и db_host как к обычным переменным.

10. Динамическая генерация group_vars и host_vars с помощью шаблонов

Вы можете хранить в репозитории шаблоны переменных (Jinja2) и в пайплайне рендерить их с использованием переменных GitLab, а затем размещать в каталогах group_vars или host_vars.

Структура репозитория

```
inventories/prod/
├── group_vars/
│   ├── all.yml.j2
│   └── web.yml.j2
└── host_vars/
    └── db01.yml.j2
```

group_vars/all.yml.j2

```yaml
app_version: {{ APP_VERSION | default('1.0.0') }}
environment: {{ CI_ENVIRONMENT_NAME | default('development') }}
```

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  script:
    - |
      for f in inventories/prod/group_vars/*.j2; do
        out="${f%.j2}"
        envsubst < "$f" > "$out"
      done
      for f in inventories/prod/host_vars/*.j2; do
        out="${f%.j2}"
        envsubst < "$f" > "$out"
      done
    - ansible-playbook -i inventories/prod playbook.yml
```

Таким образом, переменные GitLab подставляются прямо в файлы переменных Ansible.

11. Использование include_vars с файлом, созданным в пайплайне

Вместо передачи через --extra-vars, можно записать YAML-файл в пайплайне и затем включить его в плейбуке с помощью include_vars.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  script:
    - |
      cat > ci_vars.yml << EOF
      app_version: "$APP_VERSION"
      db_password: "$DB_PASSWORD"
      feature_flags:
        - new_ui
        - beta_api
      EOF
    - ansible-playbook -i inventory/prod playbook.yml
```

playbook.yml

```yaml
- hosts: all
  tasks:
    - name: Включить переменные CI
      ansible.builtin.include_vars: ci_vars.yml
  roles:
    - myapp
```

Это позволяет передавать сложные структуры (словари, списки) без проблем экранирования.

12. Использование ansible.builtin.template для генерации файла переменных на целевой машине

Если нужно передать переменные не только в процессе выполнения, но и сохранить их на целевом хосте, можно использовать шаблон.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  script:
    - ansible-playbook -i inventory/prod playbook.yml -e "app_version=$APP_VERSION db_password=$DB_PASSWORD"
```

playbook.yml

```yaml
- hosts: all
  tasks:
    - name: Создать файл с переменными приложения
      ansible.builtin.template:
        src: app_env.j2
        dest: /etc/myapp/env
        mode: '0644'
```

templates/app_env.j2

```jinja2
APP_VERSION={{ app_version }}
DB_PASSWORD={{ db_password }}
```

13. Использование GitLab CI parallel:matrix для запуска с разными переменными

Если вам нужно выполнить один и тот же плейбук с разными наборами переменных (например, для разных окружений), можно использовать матрицу параллельных задач.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  parallel:
    matrix:
      - ENVIRONMENT: [staging, production]
        APP_VERSION: ["1.0.0", "1.1.0"]
  script:
    - ansible-playbook -i inventory/$ENVIRONMENT playbook.yml --extra-vars "app_version=$APP_VERSION"
```

При этом GitLab создаст несколько параллельных задач, в каждой из которых будут свои переменные. Это удобно для тестирования или деплоя на несколько сред.

14. Использование ansible-pull с файлом конфигурации, созданным из GitLab

В сценариях, где Ansible запускается на целевых машинах (pull-режим), можно передать переменные, разместив их в файле на хосте, который обновляется через CI (например, по SSH или через систему управления конфигурациями).

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  script:
    - ssh user@target "echo 'app_version: $APP_VERSION' > /etc/ansible/facts.d/myapp.fact"
    - ssh user@target "ansible-pull -U https://gitlab.com/your/repo.git playbook.yml"
```

Ansible автоматически подхватит локальные факты из /etc/ansible/facts.d/*.fact и сделает их доступными как ansible_local.myapp.app_version.

15. Использование ansible-config и переменных окружения для настройки поведения

Некоторые параметры Ansible можно задавать через переменные окружения, которые GitLab CI передаёт автоматически. Например, ANSIBLE_HOST_KEY_CHECKING, ANSIBLE_FORKS, ANSIBLE_BECOME_PASSWORD и т.д.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  variables:
    ANSIBLE_HOST_KEY_CHECKING: "False"
    ANSIBLE_FORKS: "50"
    ANSIBLE_BECOME_PASSWORD: "$BECOME_PASSWORD"
  script:
    - ansible-playbook -i inventory/prod playbook.yml
```

Эти переменные влияют на поведение Ansible без необходимости передавать их через --extra-vars.

16. Использование внешних хранилищ секретов (HashiCorp Vault, AWS Secrets Manager)

Вместо прямой передачи секретов через GitLab переменные, можно настроить Ansible на получение секретов из внешнего хранилища. GitLab передаёт только адрес/токен для доступа к хранилищу.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  script:
    - export VAULT_TOKEN=$VAULT_TOKEN
    - ansible-playbook -i inventory/prod playbook.yml
```

В плейбуке или роли

```yaml
- hosts: all
  vars:
    db_password: "{{ lookup('hashi_vault', 'secret/data/myapp/db:password') }}"
  roles:
    - myapp
```

Здесь GitLab используется только для запуска пайплайна и передачи токена доступа к Vault, а сами секреты не попадают в логи CI.

17. Передача переменных через inventory-файл с подстановкой из GitLab

Можно генерировать inventory прямо в пайплайне с включением переменных для конкретных хостов или групп.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  script:
    - |
      echo "[web]" > inventory/prod
      echo "web01 ansible_host=$WEB01_IP app_port=$APP_PORT" >> inventory/prod
      echo "web02 ansible_host=$WEB02_IP app_port=$APP_PORT" >> inventory/prod
    - ansible-playbook -i inventory/prod playbook.yml
```

Теперь переменная app_port будет определена для каждого хоста в группе web, и роль сможет её использовать.

18. Использование --extra-vars с JSON-строкой

Если нужно передать сложную структуру (словарь или список) через --extra-vars, можно использовать JSON.

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  script:
    - |
      EXTRA_VARS='{"app_config": {"port": 8080, "ssl": true}, "tags": ["frontend", "public"]}'
      ansible-playbook -i inventory/prod playbook.yml --extra-vars "$EXTRA_VARS"
```

В плейбуке:

```yaml
- hosts: all
  tasks:
    - debug:
        msg: "Port: {{ app_config.port }}, SSL: {{ app_config.ssl }}"
```

Это удобно для передачи структурированных данных, не прибегая к созданию файлов.

19. Использование ansible.builtin.command для запуска вложенного ansible с переменными

Иногда нужно выполнить один плейбук, который внутри себя запускает другой с переменными. Переменные GitLab можно передать во вложенный вызов.

playbook.yml (основной)

```yaml
- hosts: localhost
  tasks:
    - name: Запустить вложенный плейбук
      ansible.builtin.command:
        cmd: ansible-playbook nested_playbook.yml --extra-vars "target={{ target }}"
      environment:
        TARGET: "{{ lookup('env', 'TARGET') }}"
```

.gitlab-ci.yml

```yaml
deploy:
  stage: deploy
  variables:
    TARGET: production
  script:
    - ansible-playbook main_playbook.yml
```

Это менее распространённый, но возможный вариант для сложных сценариев.

Заключение

Выбор метода зависит от архитектуры вашего пайплайна, требований безопасности, сложности переменных и используемых инструментов. Для простых случаев достаточно --extra-vars, для сложных структур — генерация файлов, для централизованного управления — Tower/AWX, для секретов — внешние хранилища. Комбинируйте подходы в зависимости от задач.