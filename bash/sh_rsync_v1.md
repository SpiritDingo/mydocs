Проверка изменений при копировании через rsync

Есть три уровня контроля. Разберу по порядку — от простого к надёжному.

1. Проверка «до/после» через --dry-run и --checksum

Что изменилось ПОСЛЕ последнего копирования

```bash
# Показать только список файлов, которые будут изменены (без реальной передачи)
rsync -avz --dry-run --checksum \
    -e "ssh -p 22" \
    /data/app1/ backupuser@192.168.1.100:/backup/data/app1/ \
    | tee /var/log/rsync_transfer/preview_$(date +%F).log
```

Флаг --checksum заставляет сравнивать по контрольной сумме (MD5), а не по времени+размеру. Это точнее, но медленнее.

Только имена изменённых файлов (без лишнего вывода)

```bash
rsync -az --dry-run --checksum --out-format='%n' \
    -e "ssh -p 22" \
    /data/app1/ backupuser@192.168.1.100:/backup/data/app1/
```

Формат %n печатает только путь. Другие полезные поля:

· %i — тип изменения (>f новый, >f+++++++++ создан, cd смена каталога, .d..t изменено время)
· %l — размер
· %M — время модификации

2. Обнаружение файлов, изменившихся ВО ВРЕМЯ копирования

Это самая частая проблема: файл в источнике меняется, пока rsync его читает → на приёмник попадает «рваная» копия. Контроль:

```bash
# Снимок mtime до
find /data/app1 -type f -printf '%T@ %p\n' | sort > /tmp/before.txt

rsync -avz --checksum ...  # само копирование

# Снимок mtime после
find /data/app1 -type f -printf '%T@ %p\n' | sort > /tmp/after.txt

# Файлы, изменившиеся за время копирования
diff /tmp/before.txt /tmp/after.txt
```

Если diff что-то показал — эти файлы нужно перенести повторно (они «плыли» под rsync).

3. Верификация целостности ПОСЛЕ копирования

Самый надёжный способ — сравнить контрольные суммы источника и приёмника.

Через rsync (встроенная проверка)

```bash
# -n = dry-run, -c = checksum, но всё равно идёт чтение с обеих сторон
rsync -navc --checksum \
    -e "ssh -p 22" \
    /data/app1/ backupuser@192.168.1.100:/backup/data/app1/ \
    | tee /var/log/rsync_transfer/verify_$(date +%F_%H%M).log
```

Если вывод пуст — всё совпадает. Если есть строки — там расхождения.

Через md5sum + diff (независимая проверка)

```bash
# На источнике
cd /data/app1 && find . -type f -exec md5sum {} + | sort -k2 > /tmp/src_sums.txt

# На приёмнике
ssh -p 22 backupuser@192.168.1.100 \
    "cd /backup/data/app1 && find . -type f -exec md5sum {} + | sort -k2" \
    > /tmp/dst_sums.txt

# Сравнение
diff /tmp/src_sums.txt /tmp/dst_sums.txt && echo "OK: всё совпадает"
```

Готовый скрипт с проверкой изменений

Вот расширение предыдущего скрипта — добавляет проверку во время и после:

```bash
#!/bin/bash
# rsync_with_verify.sh — копирование с контролем изменений

SOURCE="/data/app1/"
REMOTE_USER="backupuser"
REMOTE_HOST="192.168.1.100"
REMOTE_PORT="22"
REMOTE_DIR="/backup/data/app1/"
LOG_DIR="/var/log/rsync_transfer"
STAMP=$(date +%Y%m%d_%H%M%S)
LOG="${LOG_DIR}/rsync_${STAMP}.log"
SSH_OPTS="-p ${REMOTE_PORT} -o ConnectTimeout=30 -o StrictHostKeyChecking=accept-new"

mkdir -p "${LOG_DIR}"
log() { echo "[$(date '+%F %T')] $*" | tee -a "${LOG}"; }

log "=== СТАРТ $(date) ==="

# --- 1. Снимок состояния источника ДО ---
log "Снимок mtime источника ДО..."
find "${SOURCE}" -type f -printf '%T@ %s %p\n' | sort > "/tmp/src_before_${STAMP}.txt"
BEFORE_COUNT=$(wc -l < "/tmp/src_before_${STAMP}.txt")
log "Файлов в источнике: ${BEFORE_COUNT}"

# --- 2. Основное копирование ---
log "Запуск rsync..."
rsync -avz --partial --progress --stats \
    --numeric-ids \
    --out-format='%i %n' \
    -e "ssh ${SSH_OPTS}" \
    "${SOURCE}" "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}" &>>"${LOG}"
RC=$?

if [[ ${RC} -ne 0 ]]; then
    log "ОШИБКА: rsync вернул код ${RC}"
    exit ${RC}
fi
log "rsync завершён успешно."

# --- 3. Снимок состояния источника ПОСЛЕ ---
log "Снимок mtime источника ПОСЛЕ..."
find "${SOURCE}" -type f -printf '%T@ %s %p\n' | sort > "/tmp/src_after_${STAMP}.txt"

# --- 4. Файлы, изменившиеся во время копирования ---
CHANGED="/tmp/changed_${STAMP}.txt"
comm -13 "/tmp/src_before_${STAMP}.txt" "/tmp/src_after_${STAMP}.txt" > "${CHANGED}"
CHANGED_COUNT=$(wc -l < "${CHANGED}")

if [[ ${CHANGED_COUNT} -gt 0 ]]; then
    log "ВНИМАНИЕ: ${CHANGED_COUNT} файлов изменились во время копирования:"
    cat "${CHANGED}" | tee -a "${LOG}"
    log "Запуск повторного копирования для этих файлов..."

    while IFS= read -r LINE; do
        # из строки "mtime size path" достаём путь
        FPATH=$(echo "${LINE}" | awk '{ $1=""; $2=""; sub(/^  */,""); print }')
        if [[ -f "${FPATH}" ]]; then
            REL="${FPATH#${SOURCE}}"
            log "  Повтор: ${REL}"
            rsync -az --partial \
                -e "ssh ${SSH_OPTS}" \
                "${FPATH}" "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}$(dirname "${REL}")/" \
                &>>"${LOG}"
        fi
    done < "${CHANGED}"
    log "Повторное копирование завершено."
else
    log "Файлы не менялись во время копирования."
fi

# --- 5. Верификация контрольными суммами ---
log "Верификация контрольных сумм (может занять время)..."
rsync -navc --checksum --out-format='%n' \
    -e "ssh ${SSH_OPTS}" \
    "${SOURCE}" "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}" \
    > "/tmp/verify_${STAMP}.txt" 2>>"${LOG}"

DIFF_COUNT=$(wc -l < "/tmp/verify_${STAMP}.txt")
if [[ ${DIFF_COUNT} -eq 0 ]]; then
    log "✓ Верификация пройдена: источник и приёмник идентичны."
else
    log "✗ ОБНАРУЖЕНЫ РАСХОЖДЕНИЯ (${DIFF_COUNT} файлов):"
    cat "/tmp/verify_${STAMP}.txt" | tee -a "${LOG}"
    exit 3
fi

log "=== ГОТОВО $(date) ==="
```

Пояснения по ключевым приёмам

Почему -c (--checksum) не всегда хорош по умолчанию:
Он читает оба конца полностью и считает MD5 каждого файла — это в разы медленнее, чем сравнение mtime+size. Используйте -c:

· для финальной верификации (-navc — dry-run + checksum);
· когда время файлов на серверах «сбито» или не вызывает доверия.

Почему mtime+size из find — удобный детектор «плавающих» файлов:
Если приложение пишет в файл во время копирования, mtime сдвинется. Сравнение «до/после» по mtime мгновенно это покажет. Если и размер поменялся — файл точно нужно перелить.

rsync --out-format='%i' — расшифровка символов:

· .d..t...... — каталог, изменилось только время
· >f+++++++++ — новый файл
· >f.st...... — файл, изменился размер и время
· *deleting — удалён на приёмнике (при --delete)
· cd+++++++++ — создан каталог

Для живой верификации больших файлов (когда полный checksum дорого):

```bash
rsync -avz --partial -T /tmp/rsync_tmp \
    --info=progress2 --info=name0 \
    ...
```

--info=progress2 показывает общий прогресс — по нему видно, «застряла» ли передача на конкретном файле.

Рекомендуемая последовательность для продакшена

```bash
# 1. Предпросмотр (ничего не меняем)
rsync -avn --checksum --out-format='%i %n' ... | tee preview.log

# 2. Основное копирование
rsync -avz --partial --numeric-ids ... 

# 3. Повторный прогон (добьёт «плавающие» файлы)
rsync -avz --checksum ... 

# 4. Финальная верификация
rsync -navc --checksum ... | tee verify.log
# пусто = ОК
```

Шаги 3 и 4 и есть «проверка изменения при копировании» — они гарантируют, что всё, что менялось во время передачи, в итоге скопировано корректно.