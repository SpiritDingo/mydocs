Конечно! Ниже представлен пример **регламента использования Ansible в организации**. Этот документ может быть адаптирован под конкретные потребности вашей команды, инфраструктуру и политики безопасности.

---

# Регламент использования Ansible в организации

**Версия:** 1.0  
**Дата вступления в силу:** [указать дату]  
**Ответственный за поддержку:** [Имя/Должность/Команда, например, DevOps-команда]

---

## 1. Общие положения

1.1. Настоящий регламент определяет правила, стандарты и процедуры использования системы автоматизации конфигураций **Ansible** в рамках ИТ-инфраструктуры организации.

1.2. Цель регламента — обеспечить:
- единообразие и предсказуемость управления инфраструктурой;
- безопасность и соответствие политикам информационной безопасности;
- воспроизводимость и аудируемость изменений;
- упрощение сопровождения и передачи знаний между сотрудниками.

1.3. Регламент обязателен для всех сотрудников, подрядчиков и сторонних специалистов, имеющих доступ к инфраструктуре и использующих Ansible.

---

## 2. Архитектура и размещение кода

2.1. Все Ansible-проекты (playbooks, roles, inventory и т.д.) должны храниться в централизованном репозитории системы контроля версий (например, GitLab/GitHub/Bitbucket).

2.2. Для каждого проекта (или группы связанных проектов) создаётся отдельный репозиторий или подкаталог с понятной структурой.

2.3. Рекомендуется использовать стандартную структуру проекта по [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html).

2.4. Запрещено хранить чувствительные данные (пароли, ключи, токены) в открытом виде в репозитории.

---

## 3. Управление секретами

3.1. Все секреты (пароли, SSH-ключи, API-токены и т.д.) должны шифроваться с использованием **Ansible Vault**.

3.2. Ключи шифрования (vault password) хранятся в защищённом менеджере секретов (например, HashiCorp Vault, AWS Secrets Manager, CyberArk) и недоступны напрямую разработчикам.

3.3. При локальной разработке допускается использование временных vault-файлов с тестовыми данными, но они **не должны попадать в основную ветку** репозитория.

3.4. Доступ к секретам предоставляется по принципу **минимальных привилегий** и только по запросу с обоснованием.

---

## 4. Версионирование и ветвление

4.1. Основная рабочая ветка — `main` (или `master`). В неё разрешено мержить только прошедшие проверку изменения.

4.2. Все изменения вносятся через **feature-ветки** и **merge/pull request** с обязательным код-ревью как минимум от одного коллеги.

4.3. Для production-окружения используется тегированная версия (например, `v1.2.3`) или отдельная ветка (например, `production`), синхронизируемая по расписанию или по запросу.

---

## 5. Тестирование и валидация

5.1. Все playbooks и роли должны быть покрыты тестами (например, с использованием **Molecule**, **Testinfra** или **pytest**).

5.2. Перед мержем в основную ветку запускается CI-пайплайн, включающий:
- синтаксическую проверку (`ansible-playbook --syntax-check`);
- линтинг (например, `ansible-lint`);
- unit/integration-тесты;
- проверку безопасности (например, `bandit`, `gitleaks`).

5.3. Запрещено запускать не протестированные playbooks в production-среде.

---

## 6. Запуск и выполнение

6.1. Запуск Ansible в production-среде возможен **только через CI/CD-систему** (например, GitLab CI, Jenkins, AWX/Ansible Tower), за исключением экстренных случаев.

6.2. В экстренных случаях (например, сбой инфраструктуры) разрешён ручной запуск с обязательной последующей фиксацией изменений в репозитории и пост-мортем-анализом.

6.3. Все запуски должны логироваться, а результаты — сохраняться (например, через Ansible Tower/AWX или кастомный логгер).

6.4. Использование флага `--check` (dry-run) рекомендуется перед любым изменением в production.

---

## 7. Документирование

7.1. Каждый playbook и роль должен сопровождаться файлом `README.md`, описывающим:
- назначение;
- входные параметры (vars);
- зависимости;
- примеры использования.

7.2. Изменения в инфраструктуре фиксируются в changelog или через комментарии в merge request.

---

## 8. Безопасность и соответствие

8.1. Все Ansible-хосты должны использовать **SSH-ключи** вместо паролей (где возможно).

8.2. Учётные записи для запуска Ansible должны иметь **минимально необходимые права** (принцип least privilege).

8.3. Запрещено использование `become: yes` без явного обоснования и ревью.

8.4. Все изменения в инфраструктуре подлежат аудиту и должны быть воспроизводимы из кода.

---

## 9. Ответственность

9.1. Владельцем каждого Ansible-проекта назначается ответственное лицо (или команда), отвечающее за его поддержку, актуальность и безопасность.

9.2. Нарушение регламента может повлечь за собой ограничение доступа к инфраструктуре и дисциплинарные меры в соответствии с внутренними правилами организации.

---

## 10. Пересмотр регламента

10.1. Настоящий регламент пересматривается не реже одного раза в год или по мере изменения требований к инфраструктуре, безопасности или процессам.

10.2. Предложения по изменению регламента могут вносить любые сотрудники через внутреннюю систему трекинга задач.

---

**Утверждено:**  
[ФИО, Должность, Подпись]  
[Дата]

---

Если вы используете **Ansible Tower / AWX**, можно добавить отдельный раздел по управлению через веб-интерфейс, ролям пользователей, шаблонам заданий и т.д.

Хотите, чтобы я адаптировал этот регламент под конкретную инфраструктуру (например, облако AWS, Kubernetes, гибридную среду) или добавил шаблоны CI/CD?

Конечно! Ниже дополнение к регламенту — **раздел 11: Структура ролей и плейбуков**, который детализирует рекомендуемую организацию кода в соответствии с best practices Ansible и внутренними стандартами организации.

---

## 11. Структура ролей и плейбуков

### 11.1. Общие принципы

11.1.1. Все автоматизации должны быть реализованы с использованием **ролей** (roles), а не монолитных плейбуков, за исключением простых или разовых задач.

11.1.2. Роли должны быть **идемпотентными**, **модульными** и **повторно используемыми** в разных окружениях (dev, stage, prod).

11.1.3. Запрещено дублирование логики между ролями. При необходимости — выносить общую функциональность в отдельные shared-роли или коллекции.

---

### 11.2. Структура проекта (рекомендуемая)

Проект должен следовать стандартной структуре по [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html):

```
ansible-project/
├── inventories/
│   ├── production/
│   │   ├── hosts.ini
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   ├── webservers.yml
│   │   │   └── databases.yml
│   │   └── host_vars/
│   │       └── db01.yml
│   └── staging/
│       └── ...
├── playbooks/
│   ├── site.yml                 # Главный плейбук
│   ├── webservers.yml
│   └── databases.yml
├── roles/
│   ├── common/
│   ├── nginx/
│   ├── postgresql/
│   └── monitoring/
├── collections/
│   └── requirements.yml         # При использовании внешних коллекций
├── group_vars/
│   └── all.yml                  # Глобальные переменные (если не в inventories/)
├── host_vars/
├── ansible.cfg
├── requirements.yml             # Зависимости ролей (Ansible Galaxy)
├── .ansible-lint
├── .gitignore
└── README.md
```

> **Примечание:** Для небольших проектов допускается упрощённая структура, но с сохранением логического разделения.

---

### 11.3. Структура роли

Каждая роль должна соответствовать стандартной структуре Ansible:

```
roles/<role_name>/
├── defaults/
│   └── main.yml                 # Значения по умолчанию (низкий приоритет)
├── vars/
│   └── main.yml                 # Переменные роли (высокий приоритет)
├── tasks/
│   └── main.yml                 # Основной список задач
├── handlers/
│   └── main.yml                 # Обработчики (notify)
├── templates/
│   └── *.j2                     # Jinja2-шаблоны конфигурационных файлов
├── files/
│   └── *                        # Статические файлы для копирования
├── meta/
│   └── main.yml                 # Метаданные (зависимости, поддерживаемые ОС)
├── tests/
│   ├── inventory
│   └── test.yml                 # Пример запуска для тестирования
└── README.md                    # Описание роли, параметры, примеры
```

**Требования к содержимому:**

- В `defaults/main.yml` — только безопасные значения по умолчанию.
- В `vars/main.yml` — значения, которые не предполагается переопределять извне.
- Все пути к файлам и шаблонам — относительные (Ansible автоматически ищет в `files/` и `templates/`).
- В `meta/main.yml` обязательно указывать:
  - Поддерживаемые ОС (`galaxy_info.platforms`);
  - Зависимости от других ролей (если есть);
  - Минимальную версию Ansible.

---

### 11.4. Плейбуки

11.4.1. Плейбуки должны быть **декларативными** и **ориентированными на инвентарь**.

11.4.2. Использовать включения (`import_playbook`, `include_role`) для модульности:

```yaml
# playbooks/site.yml
- name: Configure common settings
  hosts: all
  roles:
    - common

- name: Deploy web servers
  hosts: webservers
  roles:
    - nginx
    - monitoring
```

11.4.3. Запрещено жёстко прописывать хосты или переменные внутри плейбука. Всё — через inventory или внешние vars.

11.4.4. Для сложных сценариев (например, blue/green deployment) допускается использование блоков `block`, `rescue`, `always`.

---

### 11.5. Переменные и приоритеты

11.5.1. Следовать [официальной иерархии приоритетов переменных Ansible](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence).

11.5.2. Рекомендуемый порядок задания переменных (от низкого к высокому приоритету):
- `roles/<role>/defaults/main.yml`
- `group_vars/`, `host_vars/`
- `inventories/<env>/group_vars/`
- Extra vars (`-e` или через CI/CD)

11.5.3. Использовать **префиксы** для переменных роли, чтобы избежать коллизий:
```yaml
# Хорошо
nginx_port: 80
nginx_worker_connections: 1024

# Плохо
port: 80  # неочевидно, к чему относится
```

---

### 11.6. Коллекции и внешние зависимости

11.6.1. Внешние роли и коллекции подключаются через файл `requirements.yml`:

```yaml
# requirements.yml
collections:
  - name: community.general
  - name: amazon.aws

roles:
  - src: geerlingguy.docker
    version: 3.4.0
```

11.6.2. Установка зависимостей выполняется командой:
```bash
ansible-galaxy install -r requirements.yml --roles-path ./roles
ansible-galaxy collection install -r requirements.yml
```

11.6.3. Версии зависимостей **фиксировать явно** (никаких `latest` в production).

---

### 11.7. Именование

11.7.1. Имена ролей, плейбуков, переменных — на **английском языке**, в **snake_case**.

11.7.2. Имена задач — понятные, в повелительном наклонении:
```yaml
- name: Install nginx package
  apt:
    name: nginx
    state: present
```

11.7.3. Избегать обобщённых имён (`config.yml`, `setup.yml`) — использовать конкретику (`configure_nginx.yml`, `install_postgres.yml`).

---

Этот раздел можно включить в основной регламент как **пункт 11** или как приложение «A. Стандарты оформления Ansible-кода».

Если вы используете **Ansible Collections** (а не только роли), можно добавить отдельный подраздел про структуру коллекций. Также можно приложить **чек-лист ревью** для merge request’ов.

Нужен шаблон `.ansible-lint`, пример `ansible.cfg` или CI-пайплайн для проверки ролей?

Отлично! Ниже вы найдёте:

1. **Шаблон `.ansible-lint`** — конфигурация линтера для соблюдения стандартов кода.  
2. **Пример `ansible.cfg`** — централизованная конфигурация Ansible.  
3. **Пример CI/CD-пайплайна** (на основе GitLab CI, но легко адаптируется под GitHub Actions/Jenkins) для автоматической проверки ролей и плейбуков.

Все файлы соответствуют best practices и могут быть включены в ваш регламент как приложения.

---

## 1. Шаблон `.ansible-lint`

Файл `.ansible-lint` размещается в корне репозитория и настраивает поведение `ansible-lint`.

```yaml
# .ansible-lint
---
# Уровень строгости: production (рекомендуется)
strict: true

# Отключение правил (только при обоснованной необходимости!)
skip_list:
  # Пример: отключить правило о дублировании задач (если используется include)
  # - 'no-duplicate-tasks'
  # - 'yaml[line-length]'  # если не хотите ограничение 120 символов

# Пути для анализа (по умолчанию — всё, но можно уточнить)
exclude_paths:
  - .git/
  - .venv/
  - tests/
  - .cache/

# Плагины (если используются кастомные)
# rulesdir: ./lint-rules/

# Формат вывода (default, codeclimate, sarif и др.)
format: default
```

> 💡 **Совет**: Не отключайте правила без веской причины. При необходимости — документируйте исключения в комментариях кода с помощью `# noqa <rule-id>`.

---

## 2. Пример `ansible.cfg`

Файл `ansible.cfg` в корне проекта обеспечивает единообразное поведение Ansible для всех участников команды и CI.

```ini
# ansible.cfg
[defaults]
# Пути
inventory = inventories/production/hosts.ini
roles_path = roles
collections_paths = collections

# Безопасность
host_key_checking = True
deprecation_warnings = False
command_warnings = False

# Читаемость вывода
stdout_callback = yaml
callback_whitelist = profile_tasks, timer

# Факты
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_fact_cache
fact_caching_timeout = 86400

# Улучшение производительности
forks = 20
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o PreferredAuthentications=publickey
control_path = %(directory)s/%%h-%%r

# Vault
vault_password_file = .vault_pass.py  # или .vault_pass (если plaintext, но не в репо!)

# Коллекции
interpreter_python = auto_silent

[inventory]
# Разрешить использование плагинов inventory (например, AWS EC2 dynamic inventory)
enable_plugins = host_list, script, auto, yaml, ini, toml

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o PreferredAuthentications=publickey
control_path_dir = ~/.ansible/cp
```

> ⚠️ **Важно**:  
> - Файл `.vault_pass.py` (или `.vault_pass`) **не должен попадать в Git**. Используйте CI-переменные или секрет-менеджеры.  
> - Для CI можно переопределять `inventory` и `vault_password_file` через переменные окружения (`ANSIBLE_INVENTORY`, `ANSIBLE_VAULT_PASSWORD_FILE`).

---

## 3. Пример CI/CD-пайплайна (GitLab CI)

Файл `.gitlab-ci.yml` — автоматическая проверка при пуше или MR.

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - validate

variables:
  ANSIBLE_FORCE_COLOR: "1"
  ANSIBLE_HOST_KEY_CHECKING: "False"
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

cache:
  paths:
    - .cache/pip
    - .venv/

before_script:
  - python -V
  - pip install virtualenv
  - virtualenv .venv
  - source .venv/bin/activate
  - pip install --upgrade pip
  - pip install ansible ansible-lint yamllint molecule[docker] testinfra

lint:ansible:
  stage: lint
  script:
    - ansible-lint --version
    - ansible-lint -v

lint:yamllint:
  stage: lint
  script:
    - yamllint -c .yamllint .

test:roles:
  stage: test
  services:
    - docker:dind
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
  script:
    - |
      for role in roles/*; do
        if [ -d "$role" ]; then
          echo "🧪 Testing role: $(basename $role)"
          cd "$role"
          if [ -f "molecule/default/molecule.yml" ]; then
            molecule test
          else
            echo "⚠️  No Molecule config — skipping"
          fi
          cd "$CI_PROJECT_DIR"
        fi
      done

validate:syntax:
  stage: validate
  script:
    - ansible-playbook --syntax-check playbooks/*.yml
```

### Дополнительно: `.yamllint` (для проверки YAML-синтаксиса)

```yaml
# .yamllint
---
extends: default

rules:
  line-length:
    max: 120
    level: warning
  braces:
    min-spaces-inside: 0
    max-spaces-inside: 1
  brackets:
    min-spaces-inside: 0
    max-spaces-inside: 1
  truthy:
    allowed-values: ["true", "false", "on", "off", "yes", "no"]
    check-keys: false
  comments:
    min-spaces-from-content: 1
```

---

## Как использовать

1. Поместите `.ansible-lint`, `ansible.cfg`, `.yamllint` в корень репозитория.
2. Настройте CI/CD (GitLab CI, GitHub Actions и т.д.) по примеру выше.
3. Обязательно добавьте в `.gitignore`:
   ```
   .vault_pass
   .vault_pass.py
   .venv/
   *.retry
   /tmp/
   ```

---

## Адаптация под другие CI

- **GitHub Actions**: используйте официальный action [`ansible-lint`](https://github.com/ansible/ansible-lint) и `molecule` через `actions/setup-python`.
- **Jenkins**: запускайте те же команды в pipeline-скрипте с виртуальным окружением.

---

Если вы используете **Ansible AWX/Tower**, можно также добавить раздел о синхронизации проектов из Git и проверке через Job Templates.

Нужны шаблоны для GitHub Actions или пример Molecule-конфигурации для роли?


Вот подробный обзор **разных вариантов организации `hosts` и `inventory` в Ansible**, которые можно использовать в зависимости от масштаба проекта, типа инфраструктуры и требований к управлению. Эти варианты можно включить в регламент как справочное приложение или использовать для выбора подходящей модели.

---

## Приложение A. Варианты структуры inventory в Ansible

### 🔹 Вариант 1. **Простой статический inventory (INI-формат)**  
**Подходит для:** небольших проектов, PoC, локальной разработки.

**Файл:** `inventory.ini`
```ini
[webservers]
web1 ansible_host=192.168.1.10
web2 ansible_host=192.168.1.11

[databases]
db1 ansible_host=192.168.1.20

[prod:children]
webservers
databases
```

✅ Плюсы:
- Простота чтения и редактирования.
- Не требует YAML-парсера.

❌ Минусы:
- Ограниченная выразительность (нельзя задавать сложные структуры переменных).
- Плохо масштабируется.

---

### 🔹 Вариант 2. **Статический inventory в YAML**  
**Подходит для:** средних проектов, когда нужна гибкость.

**Файл:** `inventory.yml`
```yaml
all:
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 192.168.1.10
          ansible_user: deploy
        web2:
          ansible_host: 192.168.1.11
    databases:
      hosts:
        db1:
          ansible_host: 192.168.1.20
          db_port: 5432
  vars:
    environment: production
```

✅ Плюсы:
- Поддержка вложенных групп, сложных переменных, списков, словарей.
- Лучше читается при большом количестве хостов.

❌ Минусы:
- Чуть выше порог входа для новичков.

---

### 🔹 Вариант 3. **Многоуровневый inventory по окружениям**  
**Подходит для:** production-сред с dev/stage/prod.

**Структура:**
```
inventories/
├── development/
│   ├── hosts.yml
│   ├── group_vars/
│   └── host_vars/
├── staging/
│   ├── hosts.yml
│   └── ...
└── production/
    ├── hosts.yml
    └── ...
```

**Запуск:**
```bash
ansible-playbook -i inventories/production/hosts.yml site.yml
```

✅ Плюсы:
- Чёткое разделение окружений.
- Возможность переиспользовать одни и те же роли с разными переменными.

❌ Минусы:
- Требует дисциплины в синхронизации структуры между окружениями.

---

### 🔹 Вариант 4. **Динамический inventory (плагины)**  
**Подходит для:** облачных сред (AWS, Azure, GCP), Kubernetes, VMware.

**Пример: AWS EC2 (`aws_ec2.yml`)**
```yaml
plugin: aws_ec2
regions:
  - eu-west-1
filters:
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
compose:
  ansible_host: public_ip_address
```

**Запуск:**
```bash
ansible-inventory -i aws_ec2.yml --graph
ansible-playbook -i aws_ec2.yml site.yml
```

✅ Плюсы:
- Автоматическое обнаружение хостов.
- Всегда актуальное состояние инфраструктуры.

❌ Минусы:
- Зависимость от API провайдера.
- Требует настройки учётных данных и кэширования.

---

### 🔹 Вариант 5. **Гибридный inventory (статика + динамика)**  
**Подходит для:** гибридных сред (часть on-prem, часть в облаке).

**Пример: `hybrid_inventory.yml`**
```yaml
plugin: constructed
strict: false
groups:
  cloud_hosts: cloud_provider is defined
  onprem_hosts: cloud_provider is not defined

compose:
  ansible_host: ip_address
```

+ отдельный статический файл с on-prem хостами.

Или: использование нескольких `-i`:
```bash
ansible-playbook -i static_hosts.ini -i aws_ec2.yml site.yml
```

✅ Плюсы:
- Гибкость для сложных архитектур.

❌ Минусы:
- Сложнее отладка и понимание источника хоста.

---

### 🔹 Вариант 6. **Inventory как код (с генерацией)**  
**Подходит для:** когда инфраструктура управляется через Terraform, Pulumi и т.п.

**Сценарий:**
1. Terraform создаёт инфраструктуру.
2. Скрипт (`terraform-inventory.py`) генерирует `inventory.json` на основе `terraform output`.
3. Ansible использует этот файл.

**Пример вывода (`inventory.json`):**
```json
{
  "webservers": {
    "hosts": ["54.123.45.67"]
  },
  "_meta": {
    "hostvars": {
      "54.123.45.67": {
        "ansible_user": "ubuntu"
      }
    }
  }
}
```

✅ Плюсы:
- Полная синхронизация между IaC и конфигурацией.
- Идеально для immutable infrastructure.

❌ Минусы:
- Требует CI-интеграции и скриптов.

---

### 🔹 Вариант 7. **Inventory через Ansible AWX/Tower**  
**Подходит для:** централизованного управления в enterprise.

- Inventory создаётся и редактируется через веб-интерфейс.
- Поддерживает статические и динамические источники.
- Интеграция с Credentials, Schedules, RBAC.

✅ Плюсы:
- Аудит, GUI, безопасность, RBAC.
- Идеально для команд.

❌ Минусы:
- Требует установки и поддержки AWX/Tower.

---

## Рекомендации по выбору

| Сценарий | Рекомендуемый вариант |
|--------|----------------------|
| Локальная разработка / демо | Вариант 1 или 2 |
| Несколько окружений (dev/stage/prod) | Вариант 3 |
| Облако (AWS, Azure и др.) | Вариант 4 |
| Гибридная инфраструктура | Вариант 5 |
| Инфраструктура через Terraform | Вариант 6 |
| Enterprise, команда >5 человек | Вариант 7 |

---

## Включение в регламент

Вы можете добавить этот обзор в регламент как:

> **Приложение Б. Варианты организации inventory**  
> Все проекты обязаны использовать один из утверждённых вариантов inventory (см. таблицу выше). Выбор обосновывается архитектурой инфраструктуры и согласовывается с DevOps-командой. Запрещено использовать несогласованные или самописные inventory-скрипты без ревью.

---

Нужны шаблоны для конкретного варианта (например, готовый `aws_ec2.yml` или `inventory.json` из Terraform)? Готов предоставить!