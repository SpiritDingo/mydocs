Bash скрипт для сбора логов с нескольких путей

Простой вариант

```bash
#!/bin/bash
# collect-logs.sh — сбор логов с нескольких путей в один архив

set -euo pipefail

# Куда складывать результат
OUTPUT_DIR="/tmp/logs_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$OUTPUT_DIR"

# Список путей с логами (можно файлы и директории)
PATHS=(
    "/var/log/syslog"
    "/var/log/nginx"
    "/var/log/mysql"
    "/opt/myapp/logs"
)

# Собираем
for path in "${PATHS[@]}"; do
    if [[ ! -e "$path" ]]; then
        echo "⚠️  Пропуск (не найдено): $path"
        continue
    fi

    # Делаем безопасное имя для назначения
    dest="$OUTPUT_DIR/$(echo "$path" | sed 's|^/||; s|/|_|g')"

    if [[ -d "$path" ]]; then
        echo "📁 Копирую директорию: $path"
        cp -a "$path" "$dest"
    else
        echo "📄 Копирую файл: $path"
        cp -a "$path" "$dest"
    fi
done

# Архивируем
ARCHIVE="${OUTPUT_DIR}.tar.gz"
tar -czf "$ARCHIVE" -C "$(dirname "$OUTPUT_DIR")" "$(basename "$OUTPUT_DIR")"
rm -rf "$OUTPUT_DIR"

echo "✅ Готово: $ARCHIVE"
echo "   Размер: $(du -h "$ARCHIVE" | cut -f1)"
```

Расширенный вариант (с фильтром по дате и ротацией)

```bash
#!/bin/bash
# collect-logs-advanced.sh
# Собирает логи за последние N дней

set -euo pipefail

DAYS="${1:-7}"                       # за сколько дней (по умолчанию 7)
HOSTNAME="$(hostname -s)"
STAMP="$(date +%Y%m%d_%H%M%S)"
WORK_DIR="/tmp/log-collect-${STAMP}"
ARCHIVE="/tmp/logs_${HOSTNAME}_${STAMP}.tar.gz"

mkdir -p "$WORK_DIR"

PATHS=(
    "/var/log/syslog"
    "/var/log/auth.log"
    "/var/log/nginx"
    "/var/log/mysql"
    "/var/log/journal"
    "/opt/myapp/logs"
)

echo "▶ Хост: $HOSTNAME"
echo "▶ Период: последние $DAYS дн."
echo "▶ Архив: $ARCHIVE"
echo

for path in "${PATHS[@]}"; do
    if [[ ! -e "$path" ]]; then
        echo "⚠️  Не найдено: $path"
        continue
    fi

    name="$(echo "$path" | sed 's|^/||; s|/|_|g')"
    dest="$WORK_DIR/$name"
    mkdir -p "$(dirname "$dest")"

    if [[ -d "$path" ]]; then
        # Директория — только файлы новее N дней
        echo "📁 $path"
        find "$path" -type f -mtime -"$DAYS" -print0 2>/dev/null \
            | while IFS= read -r -d '' f; do
                rel="${f#$path/}"
                mkdir -p "$dest/$(dirname "$rel")"
                cp -a "$f" "$dest/$rel"
              done
    else
        # Одиночный файл — проверяем mtime
        mtime=$(stat -c %Y "$path")
        cutoff=$(( $(date +%s) - DAYS*86400 ))
        if (( mtime >= cutoff )); then
            echo "📄 $path"
            cp -a "$path" "$dest"
        else
            echo "⏭  $path (старше $DAYS дн.)"
        fi
    fi
done

# Мета-информация
{
    echo "Host:     $HOSTNAME"
    echo "Date:     $(date -Is)"
    echo "Kernel:   $(uname -a)"
    echo "Period:   last $DAYS days"
} > "$WORK_DIR/_meta.txt"

# Архив
tar -czf "$ARCHIVE" -C "$(dirname "$WORK_DIR")" "$(basename "$WORK_DIR")"
rm -rf "$WORK_DIR"

echo
echo "✅ Архив готов: $ARCHIVE ($(du -h "$ARCHIVE" | cut -f1))"
```

Запуск

```bash
chmod +x collect-logs.sh
sudo ./collect-logs.sh          # права root нужны для /var/log
sudo ./collect-logs.sh 3        # за последние 3 дня
```

Полезные улучшения

· Удалённые хосты — обернуть в цикл по ssh user@host 'bash -s' < collect-logs.sh
· Файлы конфигов — добавить в PATHS пути типа /etc/nginx/nginx.conf
· Ротация архивов — find /tmp -name 'logs_*.tar.gz' -mtime +30 -delete
· Отправка — в конце scp/rsync на центральный сервер или curl -F file=@$ARCHIVE ...
· Исключения — добавить --exclude='*.gz' в tar, чтобы не тащить старые ротации

Скажи, куда логи должны попадать в итоге (локальный архив, scp, S3, ELK) — подгоню под задачу.