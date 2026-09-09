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