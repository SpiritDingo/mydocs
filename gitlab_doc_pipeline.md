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