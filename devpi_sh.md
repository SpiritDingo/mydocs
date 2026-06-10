Анализ скрипта

Скрипт инициализирует Devpi-сервер: настраивает пользователя root, создаёт зеркало PyPI и индексы проектов из папки /docker-entrypoint-init.d/projects/ с применением constraints из файлов constraints.txt.

Проблемы и узкие места

1. Ненадёжное ожидание
      sleep 3 не гарантирует, что сервер Devpi уже готов принимать запросы. При медленном старте возможны сбои.
2. Избыточные вызовы devpi login
      Сначала логин с пустым паролем, затем смена пароля, затем повторный логин. Можно выполнить за 1–2 шага.
3. Неэффективная проверка существования индексов
      devpi index -l | grep -q парсит весь список индексов. Быстрее попытаться получить индекс напрямую и проверить код возврата.
4. Избыточная обработка constraints
      tr '\n' ',' | sed 's/,$//' | sed 's/,/, /g' — два sed подряд. Проще заменить на paste -sd, и добавить пробелы при необходимости. Также не обрабатывается случай пустого файла.
5. Отсутствие проверки на пустой constraints.txt
      Если файл существует, но пуст, команда devpi index ... constraints="" может завершиться ошибкой.
6. Пароль в аргументах командной строки
      В процессах может быть виден пароль (ps aux). В контейнере это не критично, но лучше минимизировать.
7. Отсутствие обработки ошибок при создании индексов
      set -e остановит скрипт при любой ошибке, но некоторые сбои (например, уже существующий индекс) можно игнорировать.

Оптимизированная версия

```bash
#!/bin/bash
set -e

# Функция ожидания готовности Devpi
wait_for_devpi() {
    echo "Waiting for devpi server..."
    for i in {1..30}; do
        if devpi use http://localhost:3141 >/dev/null 2>&1; then
            echo "Devpi is ready."
            return 0
        fi
        sleep 1
    done
    echo "Error: Devpi server not available after 30 seconds."
    exit 1
}

wait_for_devpi

# Настройка аутентификации root
ROOT_PASSWORD="${DEVPISERVER_ROOT_PASSWORD:-admin123}"

# Пытаемся залогиниться с заданным паролем
if ! devpi login root --password="$ROOT_PASSWORD" 2>/dev/null; then
    # Если не получилось – возможно, пароль пустой (первый запуск)
    if devpi login root --password='' 2>/dev/null; then
        echo "Setting root password..."
        devpi user -m root password="$ROOT_PASSWORD"
        devpi login root --password="$ROOT_PASSWORD"
    else
        echo "Error: Cannot authenticate as root. Check credentials."
        exit 1
    fi
fi

# Создание зеркала PyPI (если отсутствует)
if ! devpi index root/pypi >/dev/null 2>&1; then
    devpi index -c pypi type=mirror mirror_url="https://pypi.org/simple/"
fi

# Создание индексов для проектов
PROJECTS_DIR="/docker-entrypoint-init.d/projects"
if [ -d "$PROJECTS_DIR" ]; then
    for project_dir in "$PROJECTS_DIR"/*/; do
        [ -d "$project_dir" ] || continue
        project_name=$(basename "$project_dir")
        index_name="root/${project_name}"

        # Создаём индекс, если не существует
        if ! devpi index "$index_name" >/dev/null 2>&1; then
            devpi index -c "$project_name" type=constrained bases=root/pypi volatile=False
        fi

        constraints_file="${project_dir}constraints.txt"
        if [ -f "$constraints_file" ] && [ -s "$constraints_file" ]; then
            echo "Applying constraints for $project_name..."
            # Объединяем строки в одну через запятую (без лишних пробелов)
            constraints=$(paste -sd, "$constraints_file")
            devpi index "$index_name" constraints="$constraints"
        else
            echo "No non-empty constraints.txt for $project_name, skipping."
        fi
    done
fi

echo "All project indexes configured."
```

Основные улучшения

· Динамическое ожидание – проверяем готовность Devpi каждую секунду (до 30 попыток) вместо фиксированного sleep 3.
· Экономия вызовов devpi login – сначала пробуем войти с реальным паролем, и только при неудаче – с пустым (для первого запуска).
· Быстрая проверка индексов – используем devpi index <name> и смотрим код возврата (2>/dev/null).
· Упрощённая обработка constraints – paste -sd, заменяет переводы строк на запятые, убирая лишние sed. Проверяем, что файл не пуст (-s).
· Безопасность – пароль не передаётся в echo, но всё ещё виден в аргументах devpi login (это ограничение самого devpi). Для полной защиты можно использовать DEVPISERVER_ROOT_PASSWORD через файл или переменную окружения внутри скрипта без подстановки в командную строку (но это потребует изменения логики devpi).
· Устойчивость – добавлена проверка, что project_dir действительно директория, и обработка ошибок аутентификации.

Скрипт сохраняет всю исходную логику, но работает быстрее и надёжнее.

_________

Проблема возникает из-за того, что devpi ожидает определённый формат constraints, а в файле constraints.txt могут быть пустые строки, комментарии или некорректные символы. Вот исправленная версия, которая правильно парсит и экранирует constraints:

Исправленный скрипт с корректной обработкой constraints

```bash
#!/bin/bash
set -e

# Функция ожидания готовности Devpi
wait_for_devpi() {
    echo "Waiting for devpi server..."
    for i in {1..30}; do
        if devpi use http://localhost:3141 >/dev/null 2>&1; then
            echo "Devpi is ready."
            return 0
        fi
        sleep 1
    done
    echo "Error: Devpi server not available after 30 seconds."
    exit 1
}

wait_for_devpi

# Настройка аутентификации root
ROOT_PASSWORD="${DEVPISERVER_ROOT_PASSWORD:-admin123}"

# Пытаемся залогиниться с заданным паролем
if ! devpi login root --password="$ROOT_PASSWORD" 2>/dev/null; then
    # Если не получилось – возможно, пароль пустой (первый запуск)
    if devpi login root --password='' 2>/dev/null; then
        echo "Setting root password..."
        devpi user -m root password="$ROOT_PASSWORD"
        devpi login root --password="$ROOT_PASSWORD"
    else
        echo "Error: Cannot authenticate as root. Check credentials."
        exit 1
    fi
fi

# Создание зеркала PyPI (если отсутствует)
if ! devpi index root/pypi >/dev/null 2>&1; then
    devpi index -c pypi type=mirror mirror_url="https://pypi.org/simple/"
fi

# Функция для очистки и форматирования constraints
parse_constraints() {
    local constraints_file="$1"
    local constraints_list=()
    
    # Читаем файл построчно
    while IFS= read -r line || [ -n "$line" ]; do
        # Удаляем пробелы в начале и конце
        line=$(echo "$line" | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//')
        
        # Пропускаем пустые строки и комментарии
        if [ -z "$line" ] || [[ "$line" =~ ^# ]]; then
            continue
        fi
        
        # Проверяем валидность формата (наличие оператора сравнения)
        if [[ "$line" =~ [=<>!] ]]; then
            constraints_list+=("$line")
        else
            echo "Warning: Invalid constraint format: '$line' - skipping" >&2
        fi
    done < "$constraints_file"
    
    # Объединяем через запятую (без пробелов после запятых)
    local result=$(IFS=','; echo "${constraints_list[*]}")
    echo "$result"
}

# Создание индексов для проектов
PROJECTS_DIR="/docker-entrypoint-init.d/projects"
if [ -d "$PROJECTS_DIR" ]; then
    for project_dir in "$PROJECTS_DIR"/*/; do
        [ -d "$project_dir" ] || continue
        project_name=$(basename "$project_dir")
        index_name="root/${project_name}"

        # Создаём индекс, если не существует
        if ! devpi index "$index_name" >/dev/null 2>&1; then
            devpi index -c "$project_name" type=constrained bases=root/pypi volatile=False
        fi

        constraints_file="${project_dir}constraints.txt"
        if [ -f "$constraints_file" ] && [ -s "$constraints_file" ]; then
            echo "Applying constraints for $project_name..."
            
            # Парсим constraints в правильном формате
            constraints=$(parse_constraints "$constraints_file")
            
            # Проверяем, что constraints не пустые
            if [ -n "$constraints" ]; then
                echo "Constraints: $constraints"
                devpi index "$index_name" constraints="$constraints"
            else
                echo "No valid constraints found, skipping."
            fi
        else
            echo "No non-empty constraints.txt for $project_name, skipping."
        fi
    done
fi

echo "All project indexes configured."
```

Альтернативная, более простая версия (если constraints.txt уже в правильном формате)

Если файл constraints.txt уже содержит валидные строки вида package==1.0.0 или package>=1.0.0, но проблема с пустыми строками или комментариями:

```bash
#!/bin/bash
set -e

wait_for_devpi() {
    echo "Waiting for devpi server..."
    for i in {1..30}; do
        if devpi use http://localhost:3141 >/dev/null 2>&1; then
            echo "Devpi is ready."
            return 0
        fi
        sleep 1
    done
    echo "Error: Devpi server not available after 30 seconds."
    exit 1
}

wait_for_devpi

ROOT_PASSWORD="${DEVPISERVER_ROOT_PASSWORD:-admin123}"

if ! devpi login root --password="$ROOT_PASSWORD" 2>/dev/null; then
    if devpi login root --password='' 2>/dev/null; then
        echo "Setting root password..."
        devpi user -m root password="$ROOT_PASSWORD"
        devpi login root --password="$ROOT_PASSWORD"
    else
        echo "Error: Cannot authenticate as root. Check credentials."
        exit 1
    fi
fi

if ! devpi index root/pypi >/dev/null 2>&1; then
    devpi index -c pypi type=mirror mirror_url="https://pypi.org/simple/"
fi

PROJECTS_DIR="/docker-entrypoint-init.d/projects"
if [ -d "$PROJECTS_DIR" ]; then
    for project_dir in "$PROJECTS_DIR"/*/; do
        [ -d "$project_dir" ] || continue
        project_name=$(basename "$project_dir")
        index_name="root/${project_name}"

        if ! devpi index "$index_name" >/dev/null 2>&1; then
            devpi index -c "$project_name" type=constrained bases=root/pypi volatile=False
        fi

        constraints_file="${project_dir}constraints.txt"
        if [ -f "$constraints_file" ] && [ -s "$constraints_file" ]; then
            echo "Applying constraints for $project_name..."
            
            # Удаляем комментарии и пустые строки, объединяем запятыми
            constraints=$(grep -v '^[[:space:]]*#' "$constraints_file" | \
                         grep -v '^[[:space:]]*$' | \
                         tr '\n' ',' | \
                         sed 's/,$//')
            
            if [ -n "$constraints" ]; then
                echo "Constraints: $constraints"
                devpi index "$index_name" constraints="$constraints"
            else
                echo "No valid constraints found, skipping."
            fi
        else
            echo "No non-empty constraints.txt for $project_name, skipping."
        fi
    done
fi

echo "All project indexes configured."
```

Дополнительная диагностическая версия

Для отладки, что именно вызывает ошибку 400:

```bash
# Вместо прямого вызова devpi index, сделайте:
echo "Raw constraints content:"
cat "$constraints_file"

echo "Processed constraints:"
constraints=$(grep -v '^[[:space:]]*#' "$constraints_file" | \
             grep -v '^[[:space:]]*$' | \
             tr '\n' ',' | \
             sed 's/,$//')
echo "$constraints"

# Пробуем по одному constraint'у для диагностики
IFS=',' read -ra CONSTR_ARRAY <<< "$constraints"
for constr in "${CONSTR_ARRAY[@]}"; do
    echo "Testing constraint: '$constr'"
    # Проверяем формат
    if [[ "$constr" =~ [=<>!] ]]; then
        echo "  ✓ Valid format"
    else
        echo "  ✗ Invalid format - missing operator"
    fi
done
```

Пример корректного constraints.txt

```txt
# Это комментарий
requests==2.31.0
urllib3>=2.0.0,<3.0.0
click>=8.0.0

# Пустые строки игнорируются

flask<3.0.0
```

Ключевые моменты:

· Каждая строка должна содержать оператор сравнения (==, >=, <=, >, <, !=)
· Запятые внутри версий (например, >=2.0.0,<3.0.0) допустимы
· Пробелы не должны быть вокруг запятых между пакетами
· Конечная запятая недопустима