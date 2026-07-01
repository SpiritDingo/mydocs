Преобразование Ansible-роли в этапы (stages) GitLab CI/CD позволяет запускать задачи независимо, перезапускать упавшие участки и делать пайплайн более прозрачным. Ниже — два основных подхода и практические рекомендации.

1. Анализ текущей роли Ansible

Допустим, у вас есть роль webserver с такими задачами:

```yaml
# tasks/main.yml
- name: Обновить кеш apt
  apt: update_cache=yes cache_valid_time=3600

- name: Установить Nginx
  apt: name=nginx state=present

- name: Скопировать конфиг
  template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf

- name: Перезапустить Nginx
  service: name=nginx state=restarted enabled=yes
```

Из этого можно выделить логические этапы:

· prepare — подготовка системы;
· install — установка пакетов;
· configure — применение конфигурации;
· restart — перезапуск сервиса.

2. Вариант A: полный переход на shell (без Ansible)

Каждая задача Ansible превращается в скрипт внутри GitLab CI job. Важно сохранить идемпотентность: проверяйте факт установки/наличия файла, чтобы не выполнять лишнюю работу.

```yaml
stages:
  - prepare
  - install
  - configure
  - restart

prepare:
  stage: prepare
  script:
    - apt-get update -qq

install:
  stage: install
  script:
    - dpkg -l nginx || apt-get install -y nginx
  needs: ["prepare"]

configure:
  stage: configure
  script:
    - cmp -s nginx.conf /etc/nginx/nginx.conf || cp nginx.conf /etc/nginx/nginx.conf
  needs: ["install"]

restart:
  stage: restart
  script:
    - systemctl is-active nginx && systemctl restart nginx || systemctl start nginx
  needs: ["configure"]
```

Гибкость: Stages можно запускать вручную (when: manual), перезапускать отдельно упавший job, добавлять условия (rules), параллелить независимые задачи.

3. Вариант B: использование Ansible внутри GitLab CI (рекомендуется)

Если роли уже написаны, удобнее сохранить Ansible, но разбить плейбук на этапы с помощью тегов (tags) или отдельных плейбуков.

3.1. Добавление тегов в роли

Модифицируйте tasks:

```yaml
- name: Обновить кеш apt
  apt: ...
  tags: [prepare, always]

- name: Установить Nginx
  apt: ...
  tags: [install]

- name: Скопировать конфиг
  template: ...
  tags: [configure]

- name: Перезапустить Nginx
  service: ...
  tags: [restart]
```

3.2. Stages в .gitlab-ci.yml

```yaml
variables:
  ANSIBLE_FORCE_COLOR: "true"

stages:
  - prepare
  - install
  - configure
  - restart

ansible_prepare:
  stage: prepare
  script:
    - ansible-playbook -i inventory playbook.yml --tags prepare

ansible_install:
  stage: install
  script:
    - ansible-playbook -i inventory playbook.yml --tags install
  needs: ["ansible_prepare"]

ansible_configure:
  stage: configure
  script:
    - ansible-playbook -i inventory playbook.yml --tags configure
  needs: ["ansible_install"]

ansible_restart:
  stage: restart
  script:
    - ansible-playbook -i inventory playbook.yml --tags restart
  needs: ["ansible_configure"]
```

Ещё гибче: можно вынести каждый тег в ручной режим и добавить переменную окружения для быстрого переключения.

4. Передача артефактов между stages

Если задачи генерируют файлы (сборка, шаблоны), используйте artifacts:

```yaml
build:
  stage: build
  script: make build
  artifacts:
    paths: [bin/]

deploy:
  stage: deploy
  needs: [build]
  script: ansible-playbook deploy.yml -e "artifact_path=bin/"
```

5. Динамические stages (матрица)

Для массового применения роли к разным хостам или группам:

```yaml
ansible_install:
  stage: install
  parallel:
    matrix:
      - HOST: [web1, web2, db1]
  script:
    - ansible-playbook -i inventory playbook.yml --limit $HOST --tags install
```

6. Итоговые рекомендации

· Не отказывайтесь от Ansible, если он уже написан и отлажен — просто «нарежьте» его на stages через теги.
· Stages дают визуализацию прогресса, возможность точечного рестарта и параллельного выполнения.
· Добавьте when: manual на критичные этапы (например, restart), чтобы контролировать применение.
· Для shell-реализации строго соблюдайте идемпотентность — проверяйте состояние перед выполнением команды (cmp, dpkg -l, systemctl is-active).

Таким образом, вы получаете конвейерную обработку инфраструктуры с полным контролем и прозрачностью, сохраняя все преимущества Ansible.


----

Дополним предыдущие варианты продвинутыми сценариями. Здесь собраны ещё более гибкие и мощные подходы к разбиению Ansible на стадии GitLab CI.

---

1. Динамический пайплайн на основе групп инвентаря

Если у вас десятки хостов, можно автоматически сгенерировать stage для каждой группы или даже для каждого хоста через parallel:matrix и динамическое чтение инвентаря.

```yaml
generate-matrix:
  stage: .pre
  image: alpine
  script:
    - apk add --no-cache ansible
    - ansible-inventory -i inventory --list | jq -c '{
        job: (.all.children | to_entries | map({GROUP: .key}))
      }' > matrix.json
  artifacts:
    paths: [matrix.json]

deploy:
  stage: deploy
  needs: [generate-matrix]
  parallel:
    matrix:
      - GROUP: $MATRIX_GROUP  # подставьте через скрипт чтения json или используйте includes
  script:
    - ansible-playbook -i inventory playbook.yml --limit $GROUP --tags configure,restart
```

Гибкость: Каждая группа инвентаря разворачивается параллельно, при сбое перезапускается только она.

---

2. Триггерные и дочерние пайплайны для окружений

Разделите инфраструктурные окружения (dev/stage/prod) с помощью trigger. Дочерний пайплайн может полностью наследовать шаблон, но с другими переменными.

```yaml
# .gitlab-ci.yml (родительский)
stages:
  - plan
  - deploy

dev:
  stage: deploy
  trigger:
    include: .gitlab/ci/deploy-env.yml
    strategy: depend
  variables:
    ENV: dev
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

# .gitlab/ci/deploy-env.yml (дочерний)
stages:
  - prepare
  - install
  - configure
  - restart

ansible_prepare:
  stage: prepare
  script: ansible-playbook -i inventories/$ENV playbook.yml --tags prepare
...
```

Преимущество: Полная изоляция окружений, разные правила подтверждения (например, в prod stage restart с when: manual).

---

3. Откат изменений (rollback) как отдельный stage

Добавьте специальный stage для отката. Используйте артефакты, чтобы сохранить предыдущую версию конфигурации или снапшот состояния.

```yaml
stages:
  - configure
  - verify
  - rollback

apply_config:
  stage: configure
  script:
    - cp /etc/nginx/nginx.conf backup/nginx.conf.bak
    - ansible-playbook -i inventory playbook.yml --tags configure
  artifacts:
    paths: [backup/]

rollback:
  stage: rollback
  when: manual
  script:
    - ansible-playbook -i inventory rollback.yml --tags rollback
  needs: [apply_config]
  environment:
    action: rollback
```

Сам rollback.yml может использовать copy для восстановления файла из бекапа или выполнить предыдущую версию плейбука из Git (по тегу).

---

4. Статический анализ и «холостой прогон» (dry-run)

Добавьте обязательные проверки до применения изменений, чтобы пайплайн падал раньше.

```yaml
stages:
  - lint
  - plan
  - apply

ansible_lint:
  stage: lint
  script:
    - ansible-lint playbook.yml
    - ansible-playbook -i inventory playbook.yml --syntax-check

ansible_plan:
  stage: plan
  script:
    - ansible-playbook -i inventory playbook.yml --check --diff
  artifacts:
    paths: [diff.txt]
    when: always

apply:
  stage: apply
  script:
    - ansible-playbook -i inventory playbook.yml
  when: manual
  needs: [ansible_plan]
```

Гибкость: Можно добавить rules: - if: $CI_PIPELINE_SOURCE == "merge_request_event" для запуска plan в MR, а apply – после слияния.

---

5. Интеграция с внешними секретами (HashiCorp Vault)

Ansible сам умеет читать Vault, но в CI можно передать токен и подставить через переменные GitLab.

```yaml
variables:
  VAULT_ADDR: https://vault.example.com
  VAULT_ROLE_ID: $CI_VAULT_ROLE_ID
  VAULT_SECRET_ID: $CI_VAULT_SECRET_ID

ansible_deploy:
  stage: deploy
  id_tokens:
    VAULT_JWT:
      aud: https://vault.example.com
  script:
    - export VAULT_TOKEN=$(vault write -field=token auth/gitlab-jwt/login role=ansible jwt=$VAULT_JWT)
    - ansible-playbook -i inventory playbook.yml
```

Для ещё большей гибкости можно использовать GitLab secrets с Vault.

---

6. Разделение «сборка» и «деплой» для кастомных артефактов

Если роль генерирует бинарные файлы или архивы, соберите их на отдельной стадии и передавайте как артефакт, чтобы не повторять сборку для каждого хоста.

```yaml
stages:
  - build
  - deploy

build_artifact:
  stage: build
  script:
    - make package  # создает nginx-conf.tar.gz
  artifacts:
    paths: [nginx-conf.tar.gz]

deploy:
  stage: deploy
  script:
    - ansible-playbook -i inventory playbook.yml -e "artifact=nginx-conf.tar.gz"
  needs: [build_artifact]
```

---

7. CI/CD Catalog: переиспользуемые компоненты

Создайте общий шаблон ansible-stage.yml и включайте его в проекты через include:component. Это позволит стандартизировать стадии.

```yaml
# В репозитории компонентов (gitlab-org/ci-templates/ansible)
# templates/stages.yml
spec:
  inputs:
    stage_name:
      default: deploy
    tags:
      default: "all"
---
ansible_job:
  stage: $[[ inputs.stage_name ]]
  script:
    - ansible-playbook -i inventory playbook.yml --tags $[[ inputs.tags ]]
```

В проекте:

```yaml
include:
  - component: gitlab-org/ci-templates/ansible@~latest
    inputs:
      stage_name: configure
      tags: configure
```

---

8. Параллельное выполнение на разных хостах с ручным выбором

Используйте parallel:matrix с предопределённым списком хостов, но оставьте when: manual для выборочного запуска.

```yaml
deploy_host:
  stage: deploy
  when: manual
  parallel:
    matrix:
      - HOST: [web01, web02, db01]
  script:
    - ansible-playbook -i inventory playbook.yml --limit $HOST
```

Перед запуском можно задать переменную HOST вручную через UI GitLab при старте пайплайна.

---

9. GitLab Environments и отслеживание версий

Привяжите стадии к средам GitLab, чтобы видеть историю деплоев и управлять ими.

```yaml
deploy:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  script:
    - ansible-playbook -i inventory playbook.yml -e "env=prod"
```

Добавьте on_stop для удаления окружения, если нужно.

---

10. Полная автоматизация с Terraform + Ansible

Совместите подготовку инфраструктуры и настройку. Terraform создаёт хосты, динамически генерирует inventory, и передаёт его в Ansible через артефакт.

```yaml
stages:
  - provision
  - configure

terraform:
  stage: provision
  script:
    - terraform apply -auto-approve
    - terraform output -json > tf_output.json
  artifacts:
    paths: [tf_output.json]

ansible:
  stage: configure
  script:
    - ansible-playbook -i inventory playbook.yml
  needs: [terraform]
```

Для динамического инвентаря можно использовать плагин terraform.py или формировать файл hosts на основе tf_output.json.

---

11. Автоматическая генерация .gitlab-ci.yml из Ansible

Если ролей и плейбуков много, можно написать скрипт, который анализирует задачи и создаёт готовый пайплайн. Например, Jinja2 шаблон, обходящий все роли и их теги.

```bash
ansible-playbook --list-tasks playbook.yml | grep 'tags:' | awk -F'[' '{print $2}' | tr -d ']' | tr ',' '\n' | sort -u
```

На основе этого вывода сгенерировать stages. Это можно делать в CI вручную или хранить результат.

---

12. Условное выполнение и стратегии (rules + needs)

Гибкие правила для запуска стадий на основе изменённых файлов:

```yaml
ansible_nginx:
  stage: configure
  script: ansible-playbook playbook.yml --tags nginx
  rules:
    - changes:
        - roles/nginx/**/*
    - when: manual
  needs: [ansible_prepare]
```

Теперь пайплайн запускает только то, что изменилось, экономя время.

---

Итоговая схема продвинутого пайплайна

```
.pre: генерация матрицы / проверка секретов
lint → plan (dry-run) → apply (manual, per group)
      ↓
   build artifacts → deploy to envs → healthcheck
                          ↓
                    rollback (manual)
```

Применяйте комбинацию этих вариантов в зависимости от сложности проекта. Каждый из них реализуем без отказа от Ansible, используя GitLab CI как оркестратор, а Ansible — как движок конфигурации.

