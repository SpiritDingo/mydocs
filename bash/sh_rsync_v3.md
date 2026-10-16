Скрипт с расширенным логированием и отчётом синхронизации

```bash
#!/bin/bash
#
# sync-remote.sh — синхронизация папок и файлов с удалённого сервера на локальный
# Подключение по паролю, ограничение скорости, подробные логи и отчёт.
#

set -euo pipefail

# ===================== НАСТРОЙКИ =====================

# --- Удалённый сервер ---
REMOTE_USER="user"
REMOTE_HOST="192.168.1.100"
REMOTE_PORT="22"
REMOTE_PATH="/var/www/html/"
LOCAL_PATH="/backup/html/"

# --- Аутентификация ---
REMOTE_PASSWORD="${RSYNC_PASSWORD:-ChangeMe123}"

# --- Ограничение скорости ---
BW_LIMIT="5M"

# --- Исключения ---
EXCLUDE_FILE="./exclude.txt"

# --- Логи и отчёты ---
LOG_DIR="/var/log/rsync-sync"                    # каталог для логов и отчётов
LOG_FILE="$LOG_DIR/sync.log"                     # основной лог
REPORT_DIR="$LOG_DIR/reports"                    # каталог для отчётов
LOG_RETENTION_DAYS=30                            # хранить логи N дней
REPORT_RETENTION_DAYS=90                         # хранить отчёты N дней

# --- Уведомления (опционально) ---
EMAIL_TO=""                                      # пусто = не отправлять
EMAIL_SUBJECT_PREFIX="[rsync-sync]"

# --- Прочее ---
LOCK_FILE="/tmp/rsync-sync.lock"
PRESERVE_OWNER="false"

# ===================== ГЛОБАЛЬНЫЕ ПЕРЕМЕННЫЕ ОТЧЁТА =====================

START_TIME=""
END_TIME=""
DURATION=""
STATUS="UNKNOWN"
RSYNC_STATS_RAW=""
FILES_TRANSFERRED=0
FILES_DELETED=0
BYTES_SENT="0"
BYTES_RECEIVED="0"
TOTAL_BYTES="0"
SPEED_AVG="0"
ERRORS=""

# ===================== ФУНКЦИИ ЛОГИРОВАНИЯ =====================

log() {
    local level="$1"; shift
    local ts
    ts="$(date '+%Y-%m-%d %H:%M:%S')"
    printf '[%s] [%-5s] %s\n' "$ts" "$level" "$*" | tee -a "$LOG_FILE" >&2
}

init_logs() {
    if ! mkdir -p "$LOG_DIR" "$REPORT_DIR" 2>/dev/null; then
        LOG_DIR="/tmp/rsync-sync"
        LOG_FILE="$LOG_DIR/sync.log"
        REPORT_DIR="$LOG_DIR/reports"
        mkdir -p "$LOG_DIR" "$REPORT_DIR"
    fi
    touch "$LOG_FILE"

    # Ротация логов
    find "$LOG_DIR" -maxdepth 1 -name "sync.log.*" -type f -mtime +$LOG_RETENTION_DAYS -delete 2>/dev/null || true
    find "$REPORT_DIR" -maxdepth 1 -name "report-*.txt" -type f -mtime +$REPORT_RETENTION_DAYS -delete 2>/dev/null || true

    # Если основной лог больше 10 МБ — ротация
    if [ -f "$LOG_FILE" ] && [ "$(stat -c%s "$LOG_FILE" 2>/dev/null || echo 0)" -gt 10485760 ]; then
        mv "$LOG_FILE" "$LOG_FILE.$(date '+%Y%m%d-%H%M%S')"
        touch "$LOG_FILE"
    fi
}

# ===================== СЛУЖЕБНЫЕ ФУНКЦИИ =====================

check_deps() {
    local need_sshpass="false"
    for cmd in rsync ssh; do
        command -v "$cmd" &>/dev/null || { log "ERROR" "Команда '$cmd' не найдена."; exit 1; }
    done
    [ -n "$REMOTE_PASSWORD" ] && need_sshpass="true"
    if [ "$need_sshpass" = "true" ] && ! command -v sshpass &>/dev/null; then
        log "ERROR" "Не найден 'sshpass'. Установите: apt install sshpass"
        exit 1
    fi
}

acquire_lock() {
    exec 200>"$LOCK_FILE"
    if ! flock -n 200; then
        log "WARN" "Скрипт уже запущен. Выход."
        exit 1
    fi
}

prepare_local() {
    [ -d "$LOCAL_PATH" ] || { log "INFO" "Создаю локальную папку: $LOCAL_PATH"; mkdir -p "$LOCAL_PATH"; }
}

build_ssh_cmd() {
    local ssh_opts="-p $REMOTE_PORT -o StrictHostKeyChecking=accept-new -o ConnectTimeout=15"
    if [ -n "$REMOTE_PASSWORD" ]; then
        echo "sshpass -e ssh $ssh_opts"
    else
        echo "ssh $ssh_opts"
    fi
}

build_rsync_args() {
    RSYNC_ARGS=(
        -avz
        --delete
        --partial
        --progress
        --human-readable
        --stats
        --itemize-changes
        -e "$(build_ssh_cmd)"
    )
    [ -n "$BW_LIMIT" ] && RSYNC_ARGS+=(--bwlimit="$BW_LIMIT")
    [ "$PRESERVE_OWNER" = "true" ] && RSYNC_ARGS+=(--numeric-ids --owner --group --perms)
    [ -f "$EXCLUDE_FILE" ] && RSYNC_ARGS+=(--exclude-from="$EXCLUDE_FILE")
}

# ===================== ПАРСИНГ СТАТИСТИКИ =====================

parse_rsync_stats() {
    local raw="$1"

    FILES_TRANSFERRED=$(echo "$raw" | grep -E "^Number of regular files transferred:" | awk '{print $NF}' || echo 0)
    FILES_DELETED=$(echo "$raw"     | grep -E "^Number of deleted files:"              | awk '{print $NF}' || echo 0)
    BYTES_SENT=$(echo "$raw"        | grep -E "^Total bytes sent:"                     | awk '{print $NF}' || echo 0)
    BYTES_RECEIVED=$(echo "$raw"    | grep -E "^Total bytes received:"                 | awk '{print $NF}' || echo 0)
    TOTAL_BYTES=$(echo "$raw"       | grep -E "^Total transferred file size:"          | awk '{print $NF}' || echo 0)
    SPEED_AVG=$(echo "$raw"         | grep -E "bytes/sec$"                             | tail -1 | awk '{print $(NF-1)}' || echo 0)

    : "${FILES_TRANSFERRED:=0}"
    : "${FILES_DELETED:=0}"
    : "${BYTES_SENT:=0}"
    : "${BYTES_RECEIVED:=0}"
    : "${TOTAL_BYTES:=0}"
    : "${SPEED_AVG:=0}"
}

# ===================== СИНХРОНИЗАЦИЯ =====================

do_sync() {
    local src="${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}"
    local dst="${LOCAL_PATH}"
    local raw_output

    log "INFO" "Источник : $src"
    log "INFO" "Назначение: $dst"
    log "INFO" "Скорость  : ${BW_LIMIT:-без ограничений}"

    export SSHPASS="$REMOTE_PASSWORD"

    set +e
    raw_output=$(rsync "${RSYNC_ARGS[@]}" "$src" "$dst" 2>&1)
    local rc=$?
    set -e

    # Пишем весь вывод rsync в лог
    printf '%s\n' "$raw_output" >> "$LOG_FILE"

    RSYNC_STATS_RAW="$raw_output"
    parse_rsync_stats "$raw_output"

    if [ $rc -eq 0 ]; then
        STATUS="SUCCESS"
        return 0
    elif [ $rc -eq 23 ] || [ $rc -eq 24 ]; then
        STATUS="PARTIAL"
        ERRORS="Код $rc: частичная передача (некоторые файлы пропущены/удалены)"
        return 1
    else
        STATUS="FAILED"
        ERRORS="Код $rc: $(echo "$raw_output" | grep -i 'error\|failed' | head -3 | tr '\n' ';')"
        return 1
    fi
}

# ===================== ОТЧЁТ =====================

human_bytes() {
    local b="${1:-0}"
    awk -v b="$b" 'BEGIN {
        split("B KB MB GB TB PB", u, " ");
        i=1;
        while (b >= 1024 && i < 6) { b/=1024; i++ }
        printf "%.2f %s", b, u[i]
    }'
}

generate_report() {
    local report_file="$REPORT_DIR/report-$(date '+%Y%m%d-%H%M%S').txt"

    {
        echo "============================================================"
        echo "         ОТЧЁТ О СИНХРОНИЗАЦИИ RSYNC"
        echo "============================================================"
        echo "Дата запуска    : $START_TIME"
        echo "Дата окончания  : $END_TIME"
        echo "Длительность    : $DURATION"
        echo "Статус          : $STATUS"
        echo ""
        echo "--- Параметры ---"
        echo "Источник        : ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}"
        echo "Назначение      : ${LOCAL_PATH}"
        echo "Порт SSH        : ${REMOTE_PORT}"
        echo "Огранич. скор.  : ${BW_LIMIT:-не задано}"
        echo "Исключения      : ${EXCLUDE_FILE}"
        echo ""
        echo "--- Результаты ---"
        echo "Файлов передано : $FILES_TRANSFERRED"
        echo "Файлов удалено  : $FILES_DELETED"
        echo "Всего передано  : $(human_bytes "$TOTAL_BYTES") ($TOTAL_BYTES байт)"
        echo "Отправлено      : $(human_bytes "$BYTES_SENT")"
        echo "Получено        : $(human_bytes "$BYTES_RECEIVED")"
        echo "Средняя скорость: $SPEED_AVG байт/с ($(human_bytes "$SPEED_AVG")/с)"
        echo ""
        if [ -n "$ERRORS" ]; then
            echo "--- Ошибки ---"
            echo "$ERRORS"
            echo ""
        fi
        echo "--- Статистика rsync ---"
        echo "$RSYNC_STATS_RAW" | grep -E "^(Number of|Total|sent|total size)" || true
        echo ""
        echo "--- Изменённые файлы (itemize) ---"
        echo "$RSYNC_STATS_RAW" | grep -E "^[<>ch.][fdLDS]" | head -200 || echo "(нет изменений)"
        echo ""
        echo "============================================================"
        echo "Полный лог: $LOG_FILE"
        echo "============================================================"
    } > "$report_file"

    ln -sf "$(basename "$report_file")" "$REPORT_DIR/latest.txt"

    log "INFO" "Отчёт сохранён: $report_file"
    echo "$report_file"
}

send_email_report() {
    local report_file="$1"
    [ -z "$EMAIL_TO" ] && return 0
    command -v mail &>/dev/null || { log "WARN" "mail не установлен, пропуск email."; return 0; }

    local subject="$EMAIL_SUBJECT_PREFIX $STATUS — $START_TIME"
    mail -s "$subject" "$EMAIL_TO" < "$report_file" && log "INFO" "Отчёт отправлен на $EMAIL_TO"
}

print_summary() {
    echo ""
    echo "════════════════════════════════════════════"
    echo " ИТОГ СИНХРОНИЗАЦИИ: $STATUS"
    echo "════════════════════════════════════════════"
    printf " Длительность     : %s\n" "$DURATION"
    printf " Файлов передано  : %s\n" "$FILES_TRANSFERRED"
    printf " Файлов удалено   : %s\n" "$FILES_DELETED"
    printf " Всего передано   : %s\n" "$(human_bytes "$TOTAL_BYTES")"
    printf " Средняя скорость : %s/с\n" "$(human_bytes "$SPEED_AVG")"
    [ -n "$ERRORS" ] && printf " Ошибки           : %s\n" "$ERRORS"
    echo "════════════════════════════════════════════"
    echo ""
}

# ===================== ОСНОВНОЙ БЛОК =====================

main() {
    init_logs
    START_TIME="$(date '+%Y-%m-%d %H:%M:%S')"
    log "INFO" "═════ Запуск синхронизации ═════"

    check_deps
    acquire_lock
    prepare_local
    build_rsync_args

    local rc=0
    do_sync || rc=$?

    END_TIME="$(date '+%Y-%m-%d %H:%M:%S')"
    local start_epoch end_epoch
    start_epoch=$(date -d "$START_TIME" +%s 2>/dev/null || echo 0)
    end_epoch=$(date -d "$END_TIME" +%s 2>/dev/null || echo 0)
    DURATION=$(printf '%02d:%02d:%02d' $(( (end_epoch - start_epoch)/3600 )) $(( ((end_epoch - start_epoch)%3600)/60 )) $(( (end_epoch - start_epoch)%60 )))

    local report_file
    report_file=$(generate_report)
    send_email_report "$report_file"
    print_summary

    unset SSHPASS
    log "INFO" "═════ Завершено: $STATUS ═════"
    exit $rc
}

main "$@"
```

Что добавлено

1. Расширенное логирование

· Логи пишутся в каталог /var/log/rsync-sync/ (с fallback в /tmp при отсутствии прав)
· Основной лог sync.log с автоматической ротацией при размере > 10 МБ
· Автоочистка логов старше LOG_RETENTION_DAYS (30 дней) и отчётов старше REPORT_RETENTION_DAYS (90 дней)
· В лог попадает весь вывод rsync (включая itemize-изменения файлов)

2. Формирование отчёта

Каждый запуск создаёт файл reports/report-YYYYMMDD-HHMMSS.txt с секциями:

Секция Содержимое
Общее Дата старта/окончания, длительность, статус
Параметры источник, назначение, порт, лимит скорости, исключения
Результаты файлов передано/удалено, объёмы, средняя скорость
Ошибки если были — с расшифровкой кода rsync
Статистика rsync сырые данные из --stats
Изменённые файлы список изменений (itemize-changes), до 200 строк

Также создаётся симлинк reports/latest.txt — всегда указывает на последний отчёт.

3. Человеко-читаемые размеры

Функция human_bytes конвертирует байты в КБ / МБ / ГБ / ТБ.

4. Разбор кодов возврата rsync

Код Статус
0 ✅ SUCCESS
23, 24 ⚠️ PARTIAL (частичная передача)
остальные ❌ FAILED

5. Уведомления по email (опционально)

Установите EMAIL_TO="admin@example.com" — отчёт уйдёт через mail/mailx.

6. Итоговая сводка в консоль

В конце работы выводится компактная рамка с ключевыми метриками.

Пример вывода в консоль

```
════════════════════════════════════════════
 ИТОГ СИНХРОНИЗАЦИИ: SUCCESS
════════════════════════════════════════════
 Длительность     : 00:02:14
 Файлов передано  : 47
 Файлов удалено   : 3
 Всего передано   : 128.45 MB
 Средняя скорость : 981.23 KB/с
════════════════════════════════════════════
```

Просмотр результатов

```bash
# Последний отчёт
cat /var/log/rsync-sync/reports/latest.txt

# Все отчёты за неделю
find /var/log/rsync-sync/reports/ -name 'report-*.txt' -mtime -7

# Живой лог
tail -f /var/log/rsync-sync/sync.log

# Только ошибки из лога
grep '\[ERROR\]' /var/log/rsync-sync/sync.log
```

Автоматизация через cron с уведомлением

```cron
# Каждый час, отчёт отправляется на email
0 * * * * EMAIL_TO="admin@example.com" /opt/scripts/sync-remote.sh >/dev/null 2>&1
```

Возможные дальнейшие улучшения

· Telegram-уведомления — через curl к Bot API с текстом отчёта
· HTML-отчёт — вместо .txt генерировать .html для удобного просмотра
· Проверка целостности — добавить --checksum и сравнение контрольных сумм
· Мониторинг в Grafana — писать метрики в формате Prometheus textfile
· История в SQLite — писать статистику в БД для графиков трендов

Если нужно — могу добавить HTML-отчёт, уведомления в Telegram или запись метрик в Prometheus.