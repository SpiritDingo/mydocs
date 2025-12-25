Настройка pip config

1. Расположение конфигурационных файлов pip

Linux/macOS:

· Глобальный: /etc/pip.conf или /etc/xdg/pip/pip.conf
· Пользовательский: ~/.pip/pip.conf или ~/.config/pip/pip.conf

Windows:

· Глобальный: C:\ProgramData\pip\pip.ini
· Пользовательский: %APPDATA%\pip\pip.ini или %USERPROFILE%\pip\pip.ini

2. Просмотр текущих настроек

```bash
# Показать все текущие настройки
pip config list

# Показать настройки с указанием источников
pip config list -v
```

3. Основные команды управления конфигурацией

```bash
# Установить параметр
pip config set global.index-url https://pypi.org/simple

# Удалить параметр
pip config unset global.index-url

# Редактировать конфигурационный файл
pip config edit
```

4. Примеры конфигураций

Пример 1: Смена репозитория (mirror)

```bash
# Установка зеркала (например, для Китая или России)
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/
pip config set global.trusted-host mirrors.aliyun.com

# Или через файл ~/.pip/pip.conf:
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
trusted-host = pypi.tuna.tsinghua.edu.cn
```

Пример 2: Настройка прокси

```bash
# Через команду
pip config set global.proxy http://user:pass@proxy-server:port

# Через файл:
[global]
proxy = http://10.10.1.10:3128
```

Пример 3: Таймауты и повторные попытки

```ini
[global]
timeout = 60
retries = 5
```

Пример 4: Отключение кэша или изменение его расположения

```ini
[global]
no-cache-dir = false
cache-dir = /path/to/cache/dir
```

Пример 5: Формат вывода и логирование

```ini
[global]
log = /path/to/pip.log
log-format = %(message)s
```

Пример 6: Безопасность и сертификаты

```ini
[global]
cert = /path/to/certificate.pem
trusted-host = 
    pypi.org
    files.pythonhosted.org
```

5. Полный пример конфигурационного файла

```ini
# ~/.pip/pip.conf или %APPDATA%\pip\pip.ini
[global]
# Основной репозиторий
index-url = https://pypi.org/simple

# Дополнительные репозитории
extra-index-url = 
    https://repo.example.com/simple
    https://pypi.company.com/internal

# Прокси-сервер
proxy = http://proxy.example.com:8080

# Таймауты
timeout = 30
retries = 3
default-timeout = 60

# Кэширование
cache-dir = ~/.cache/pip
no-cache-dir = false

# Безопасность
trusted-host = 
    pypi.org
    files.pythonhosted.org
    repo.example.com

# Формат вывода
log = ~/.pip/pip.log
log-format = concise

[install]
# Опции установки
ignore-installed = false
no-deps = false
pre = false
use-feature = fast-deps

[search]
index = https://pypi.org/pypi
```

6. Работа с переменными окружения

```bash
# Временная смена репозитория
export PIP_INDEX_URL=https://pypi.org/simple
export PIP_TRUSTED_HOST=pypi.org

# Windows (cmd)
set PIP_INDEX_URL=https://pypi.org/simple

# Windows (PowerShell)
$env:PIP_INDEX_URL = "https://pypi.org/simple"
```

7. Разделы конфигурации

· [global] - глобальные настройки
· [install] - настройки установки пакетов
· [freeze] - настройки команды freeze
· [search] - настройки поиска пакетов
· [wheel] - настройки wheel сборки

8. Проверка конфигурации

```bash
# Проверить, где ищется конфигурация
python -m pip config debug

# Проверить конкретную опцию
pip config get global.index-url
```

9. Удаление всех настроек

```bash
# Удалить все настройки из пользовательского файла
rm ~/.pip/pip.conf  # Linux/macOS
del %APPDATA%\pip\pip.ini  # Windows
```

10. Использование requirements.txt с конфигурацией

```bash
# requirements.txt может содержать ссылки на другие индексы
--index-url https://pypi.org/simple
--trusted-host pypi.org
--extra-index-url https://internal.example.com/simple

package1==1.0.0
package2>=2.0.0
```

Эти примеры помогут настроить pip под ваши нужды, будь то работа с приватными репозиториями, прокси-серверами или специфические требования к установке пакетов.




Подробный разбор разделов конфигурации pip

1. [global] - Глобальные настройки

Самый важный раздел, применяется ко всем командам pip.

```ini
[global]
# Основной репозиторий пакетов
index-url = https://pypi.org/simple

# Дополнительные репозитории (будут проверяться если пакет не найден в основном)
extra-index-url = 
    https://download.pytorch.org/whl/cu118
    https://internal.company.com/simple/

# Альтернативный формат для extra-index-url
extra-index-url = https://download.pytorch.org/whl/cu118
extra-index-url = https://internal.company.com/simple/

# Безопасность - доверенные хосты (для self-signed certs)
trusted-host = 
    pypi.org
    files.pythonhosted.org
    download.pytorch.org

# Таймауты (секунды)
timeout = 60                 # Таймаут скачивания
default-timeout = 120        # Таймаут по умолчанию
retries = 5                  # Количество повторных попыток
retry-delay = 2              # Задержка между повторами (секунды)

# Прокси-серверы
proxy = http://user:password@proxy.company.com:8080
# Или раздельно для http/https
# proxy.http = http://proxy:8080
# proxy.https = https://proxy:8080

# Кэширование
cache-dir = /custom/path/.cache/pip
no-cache-dir = false         # true - отключить кэш полностью

# Директории установки
target = /custom/install/path  # Куда устанавливать пакеты
user = false                   # Не использовать --user автоматически
root = /alternative/root

# Параметры HTTP
cert = /path/to/cert.pem      # Пользовательский SSL сертификат
client-cert = /path/to/client-cert.pem
```

Пример с комментом использования:

```ini
[global]
# Используем локальный devpi сервер как основной
index-url = http://localhost:3141/root/pypi/+simple/

# Но если пакета нет локально, ищем в PyPI
extra-index-url = https://pypi.org/simple

# Наш локальный сервер без SSL
trusted-host = localhost

# Долгие таймауты для медленного VPN
timeout = 120
retries = 10

# Кэшируем в домашней директории
cache-dir = ~/.pip-cache
```

2. [install] - Настройки установки

Применяется только к командам install и download.

```ini
[install]
# Поведение при установке
ignore-installed = false      # Игнорировать уже установленные версии
ignore-requires-python = false # Игнорировать ограничения версии Python
no-deps = false              # Не устанавливать зависимости
pre = true                   # Включать pre-release версии (альфа/бета/RC)

# Фичи pip (можно указывать несколько)
use-feature = 
    fast-deps                # Ускоренная установка зависимостей
    2020-resolver            # Использовать новый резолвер
    truststore               # Использовать системное хранилище сертификатов

# Стратегия разрешения зависимостей
only-binary = :all:          # Использовать только wheel-пакеты
no-binary = psycopg2         # Только для psycopg2 собирать из исходников
# или для всех:
# no-binary = :all:          # Всегда собирать из исходников

# Компиляция
compile = true               # Компилировать .py файлы в .pyc
no-compile = false

# Пути установки
prefix = /opt/python         # Префикс для установки
src = /opt/python/src        # Директория для исходников
editable = true              # Разрешить установку в режиме разработки (-e)

# Параметры сборки
build = /tmp/build           # Временная директория для сборки
build-option = -j4           # Параметры для setup.py build_ext
```

Примеры использования:

```ini
[install]
# Установка для production - только stable версии
pre = false
no-deps = false

# Используем новые фичи pip
use-feature = fast-deps

# Для этих пакетов всегда собираем из исходников
no-binary = 
    :all:grpc-   # все пакеты начинающиеся с grpc-
    tensorflow
    torch

# Ускорение сборки
build-option = 
    -j8
    --parallel=8
```

3. [wheel] - Настройки сборки wheel

Для управления созданием бинарных пакетов.

```ini
[wheel]
# Директории
build-dir = /tmp/wheel_build
cache-dir = ~/.cache/pip/wheels

# Бинарные пакеты
no-binary = :all:            # Не использовать wheel вообще
only-binary = numpy,scipy    # Только wheel для numpy и scipy

# Параметры сборки
universal = true             # Собирать universal wheel (py2.py3)
compile = false              # Не компилировать C-расширения
build-number = 1             # Номер сборки

# Теги платформы
platform = manylinux2014_x86_64
python-tag = cp39            # Тег версии Python (cp39, py3, etc.)
abi-tag = cp39m
```

Пример для CUDA-сборки:

```ini
[wheel]
# Специфичная сборка для CUDA
platform = linux_x86_64
python-tag = cp310

# Собираем из исходников, но используем wheel если есть CUDA версия
no-binary = torch,torchvision,torchaudio

# Директория для кэширования wheel
cache-dir = /home/user/.cache/pip/wheels-cuda11
```

4. [download] - Настройки скачивания

Для команды download.

```ini
[download]
# Куда скачивать
dest = ./downloads           # Директория для скачанных пакетов

# Что скачивать
no-deps = true              # Скачивать только указанный пакет
platform = manylinux1_x86_64 # Для какой платформы
python-version = 39         # 27, 33, 34, 35, 36, 37, 38, 39, 310, 311
implementation = cp         # cp (CPython), pp (PyPy), jy (Jython)
abi = cp39m                 # ABI тег

# Поведение
no-binary = :all:           # Не скачивать wheel
only-binary = pandas        # Только wheel для pandas
```

Пример для офлайн-инсталляции:

```ini
[download]
# Скачиваем все для офлайн установки
dest = /mnt/packages/offline
platform = manylinux2014_x86_64
python-version = 310
implementation = cp

# Скачиваем все зависимости
no-deps = false

# Собираем wheel локально
no-binary = :none:
```

5. [freeze] - Вывод зависимостей

Для команды freeze.

```ini
[freeze]
# Формат вывода
all = true                  # Включать все пакеты, даже не pip
exclude = 
    pip
    setuptools
    wheel

# Параметры вывода
exclude-editable = false    # Исключать editable пакеты
local = true               # Включать локальные пути

# Сортировка
sort = true                # Сортировать по имени
reverse = false
```

Пример для production:

```ini
[freeze]
# Только pip-пакеты, отсортированные
all = false
exclude = pip
sort = true

# Ограничиваем зависимости
path = ./requirements
exclude-editable = true
```

6. [search] - Поиск пакетов

Для команды search.

```ini
[search]
index = https://pypi.org/pypi  # Endpoint для поиска
timeout = 30                   # Таймаут поиска
retries = 3                    # Повторы при ошибке

# Форматирование вывода
format = columns              # columns, list, detailed
limit = 20                    # Количество результатов
```

7. [uninstall] - Удаление пакетов

```ini
[uninstall]
yes = false                  # Не спрашивать подтверждение
requirements = ./requirements.txt # Удалять по requirements файлу
no-deps = false              # Удалять зависимости
```

8. [list] - Список пакетов

```ini
[list]
# Формат вывода
format = columns             # columns, freeze, json
not-required = false         # Показывать только необязательные
outdated = false             # Показывать только устаревшие
uptodate = false             # Показывать только актуальные
```

9. [cache] - Управление кэшем

```ini
[cache]
dir = ~/.cache/pip
timeout = 86400              # Время жизни кэша (сек)
size = 1024                  # Максимальный размер (МБ)
```

10. [network] - Настройки сети

```ini
[network]
timeout = 60
retries = 5
proxy = http://proxy:8080
cert = /path/to/cert.pem
```

Полный пример профессиональной конфигурации

```ini
# ~/.pip/pip.conf
[global]
index-url = https://pypi.company.com/simple
extra-index-url = https://pypi.org/simple
trusted-host = 
    pypi.company.com
    pypi.org
    files.pythonhosted.org
timeout = 120
retries = 10
cache-dir = /shared/.pip-cache
no-cache-dir = false

[install]
pre = false
use-feature = fast-deps
no-binary = 
    :all:grpc-
    tensorflow
    opencv-python
only-binary = numpy,pandas
ignore-installed = false
compile = true
build-option = -j$(nproc)

[wheel]
platform = manylinux2014_x86_64
python-tag = cp311
build-dir = /tmp/pip-build
universal = false

[download]
dest = /mnt/package-cache
platform = manylinux2014_x86_64
python-version = 311
implementation = cp
abi = cp311

[freeze]
all = false
exclude = pip,setuptools,wheel
sort = true
format = requirements.txt

[search]
index = https://pypi.org/pypi
timeout = 30
limit = 50

[uninstall]
yes = true

[list]
format = columns
outdated = true

[cache]
dir = /shared/.pip-cache
timeout = 604800  # 7 дней
size = 5120       # 5 GB
```

Важные замечания:

1. Приоритет настроек (от высшего к низшему):
   ```
   Командная строка > Переменные окружения > Локальный pip.conf > Пользовательский pip.conf > Глобальный pip.conf
   ```
2. Переменные окружения имеют префикс PIP_:
   ```bash
   export PIP_INDEX_URL=https://pypi.org/simple
   export PIP_NO_CACHE_DIR=1
   export PIP_TIMEOUT=60
   ```
3. Специфичные настройки для проекта можно разместить в:
   · ./pip.conf в корне проекта
   · setup.cfg в секции [tool:pip]
   · pyproject.toml в секции [tool.pip]
4. Проверить активные настройки:
   ```bash
   # Показать все настройки
   pip config list
   
   # С указанием источника каждой настройки
   pip config list -v
   
   # Показать значение конкретной опции
   pip config get global.index-url
   ```

Эта детальная конфигурация позволяет тонко настроить поведение pip под любые требования проекта и инфраструктуры.