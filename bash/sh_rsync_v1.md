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

____
____

Авторизация по паролю для rsync

У rsync нет собственного параметра пароля — он использует SSH. Значит, вариантов три:

Способ Безопасность Удобство Когда использовать
sshpass ⚠️ низкая (пароль в скрипте/окружении) высокое внутренняя сеть, автоматизация
expect ⚠️ низкая среднее если sshpass недоступен
SSH-ключ с паролем + ssh-agent ✅ высокая среднее рекомендуется

Разберу все три, но сразу скажу: по-настоящему правильный способ — ключи (способ 3). Пароль в скрипте — компромисс, оправданный только в доверенной сети.

---

Вариант 1. sshpass (самый простой)

Установка

```bash
# Debian/Ubuntu
apt-get install -y sshpass

# RHEL/CentOS/Fedora
yum install -y sshpass
# или
dnf install -y sshpass
```

Безопасный способ передачи пароля (без аргумента -p)

Аргумент -p виден в ps aux всем пользователям системы. Правильно — через переменную окружения или файл:

```bash
# Через переменную окружения (пароль не виден в ps)
export SSHPASS='mypassword'
rsync -avz -e "sshpass -e ssh -p 22 -o StrictHostKeyChecking=accept-new" \
    /data/app1/ user@192.168.1.100:/backup/data/app1/

# Или из файла (chmod 600)
echo 'mypassword' > /root/.rsync_pass
chmod 600 /root/.rsync_pass
rsync -avz -e "sshpass -f /root/.rsync_pass ssh -p 22" \
    /data/app1/ user@192.168.1.100:/backup/data/app1/
```

Обновлённый скрипт с паролем

```bash
#!/bin/bash
#
# rsync_transfer_pass.sh — перенос каталогов с авторизацией по паролю
#

# ============ НАСТРОЙКИ ============
SOURCE_DIRS=(
    "/data/app1/"
    "/data/app2/"
)

REMOTE_USER="backupuser"
REMOTE_HOST="192.168.1.100"
REMOTE_PORT="22"
REMOTE_BASE_DIR="/backup/data"

# --- Авторизация ---
# Вариант A: файл с паролем (рекомендуется, chmod 600)
PASS_FILE="/root/.rsync_pass"
# Вариант B: пароль в переменной (раскомментируйте, если файла нет)
# RSYNC_PASSWORD="mypassword"

LOG_DIR="/var/log/rsync_transfer"
LOG_FILE="${LOG_DIR}/rsync_$(date +%Y%m%d_%H%M%S).log"

MAX_RETRIES=3
SSH_TIMEOUT=30

# ============ ПОДГОТОВКА ============
mkdir -p "${LOG_DIR}"
umask 077   # чтобы лог не был доступен всем

log() {
    local level="$1"; shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${level}] $*" | tee -a "${LOG_FILE}"
}

# Проверка наличия sshpass
if ! command -v sshpass &>/dev/null; then
    log "ERROR" "sshpass не установлен. Установите: apt-get install sshpass"
    exit 1
fi

# Определяем, откуда брать пароль
if [[ -f "${PASS_FILE}" ]]; then
    # Проверяем права на файл
    PERMS=$(stat -c '%a' "${PASS_FILE}")
    if [[ "${PERMS}" != "600" && "${PERMS}" != "400" ]]; then
        log "WARN" "Права на ${PASS_FILE} = ${PERMS}. Рекомендуется 600."
    fi
    SSHPASS_CMD="sshpass -f ${PASS_FILE}"
    log "INFO" "Авторизация: пароль из файла ${PASS_FILE}"
elif [[ -n "${RSYNC_PASSWORD}" ]]; then
    export SSHPASS="${RSYNC_PASSWORD}"
    SSHPASS_CMD="sshpass -e"
    log "INFO" "Авторизация: пароль из переменной окружения"
else
    log "ERROR" "Не задан пароль (${PASS_FILE} не найден и RSYNC_PASSWORD пуст)"
    exit 1
fi

# SSH-опции
SSH_OPTS="-p ${REMOTE_PORT} -o ConnectTimeout=${SSH_TIMEOUT} -o StrictHostKeyChecking=accept-new"

# ============ СТАРТ ============
log "INFO" "======================================================"
log "INFO" "Запуск переноса с авторизацией по паролю"
log "INFO" "Источник: ${SOURCE_DIRS[*]}"
log "INFO" "Назначение: ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BASE_DIR}"
log "INFO" "======================================================"

# Проверка соединения (через sshpass)
log "INFO" "Проверка соединения с ${REMOTE_HOST}..."
if ! ${SSHPASS_CMD} ssh ${SSH_OPTS} "${REMOTE_USER}@${REMOTE_HOST}" "echo ok" &>>"${LOG_FILE}"; then
    log "ERROR" "Не удалось подключиться к ${REMOTE_HOST}. Проверьте пароль."
    exit 1
fi
log "INFO" "Соединение установлено."

# Создаём базовый каталог
${SSHPASS_CMD} ssh ${SSH_OPTS} "${REMOTE_USER}@${REMOTE_HOST}" \
    "mkdir -p '${REMOTE_BASE_DIR}'" &>>"${LOG_FILE}" \
    || { log "ERROR" "Не удалось создать ${REMOTE_BASE_DIR}"; exit 1; }

# ============ ПЕРЕНОС ============
TOTAL=0; SUCCESS=0; FAILED=0
FAILED_DIRS=()

for SRC in "${SOURCE_DIRS[@]}"; do
    TOTAL=$((TOTAL + 1))

    if [[ ! -d "${SRC}" ]]; then
        log "WARN" "Источник не найден, пропуск: ${SRC}"
        FAILED=$((FAILED + 1))
        FAILED_DIRS+=("${SRC} (не существует)")
        continue
    fi

    DEST="${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BASE_DIR}/$(basename "${SRC}")"
    log "INFO" "------------------------------------------------------"
    log "INFO" "Перенос: ${SRC}  ->  ${DEST}"

    RETRY=0; RC=1
    while [[ ${RETRY} -lt ${MAX_RETRIES} && ${RC} -ne 0 ]]; do
        RETRY=$((RETRY + 1))
        log "INFO" "Попытка ${RETRY}/${MAX_RETRIES}..."

        # ВАЖНО: sshpass оборачивает именно ssh, а не rsync
        rsync -avz --delete \
            --numeric-ids \
            --partial --progress --stats --human-readable \
            -e "${SSHPASS_CMD} ssh ${SSH_OPTS}" \
            "${SRC}" "${DEST}" &>>"${LOG_FILE}"
        RC=$?

        if [[ ${RC} -eq 0 ]]; then
            log "INFO" "Успешно: ${SRC}"
            SUCCESS=$((SUCCESS + 1))
        else
            log "WARN" "Ошибка rsync (код ${RC}) для ${SRC}"
            sleep 5
        fi
    done

    if [[ ${RC} -ne 0 ]]; then
        log "ERROR" "Не удалось перенести ${SRC} после ${MAX_RETRIES} попыток"
        FAILED=$((FAILED + 1))
        FAILED_DIRS+=("${SRC}")
    fi
done

# ============ ИТОГИ ============
log "INFO" "======================================================"
log "INFO" "ИТОГИ: всего=${TOTAL}, успешно=${SUCCESS}, ошибок=${FAILED}"
for D in "${FAILED_DIRS[@]}"; do
    log "ERROR" "  - Не перенесено: ${D}"
done
log "INFO" "Лог сохранён: ${LOG_FILE}"

[[ ${FAILED} -eq 0 ]] && exit 0 || exit 2
```

Настройка перед запуском

```bash
# Создаём файл с паролем
echo 'mypassword' > /root/.rsync_pass
chmod 600 /root/.rsync_pass

# Запуск
chmod +x rsync_transfer_pass.sh
sudo ./rsync_transfer_pass.sh
```

Ключевой момент: sshpass оборачивает не rsync, а ssh:

```bash
-e "sshpass -f /root/.rsync_pass ssh -p 22 ..."
```

Если написать sshpass ... rsync, работать не будет — rsync запускает ssh сам.

---

Вариант 2. expect (если sshpass недоступен)

```bash
#!/usr/bin/expect -f
# rsync_expect.exp

set timeout 3600
set password "mypassword"
set src "/data/app1/"
set dst "backupuser@192.168.1.100:/backup/data/app1/"

spawn rsync -avz --partial -e "ssh -p 22 -o StrictHostKeyChecking=accept-new" $src $dst

expect {
    "password:" {
        send "$password\r"
        exp_continue
    }
    "Password:" {
        send "$password\r"
        exp_continue
    }
    "yes/no" {
        send "yes\r"
        exp_continue
    }
    eof
}

catch wait result
exit [lindex $result 3]
```

Запуск:

```bash
chmod +x rsync_expect.exp
./rsync_expect.exp | tee /var/log/rsync_transfer/rsync_$(date +%F).log
```

Минус: пароль в открытом виде в файле скрипта, плюс expect капризен с локалями.

---

Вариант 3. SSH-ключ с паролем + ssh-agent (рекомендуемый)

Позволяет и пароль сохранить, и безопасность не терять: ключ на диске зашифрован, но в рамках сессии ssh-agent держит его расшифрованным.

Разовая настройка

```bash
# 1. Создаём ключ С паролем (passphrase)
ssh-keygen -t ed25519 -f /root/.ssh/id_rsa_rsync -C "rsync-backup"
#   введёте passphrase — это и есть "пароль"

# 2. Копируем публичный ключ на приёмник
ssh-copy-id -i /root/.ssh/id_rsa_rsync.pub -p 22 backupuser@192.168.1.100
#   здесь ОДИН РАЗ введёте пароль пользователя backupuser
```

Запуск скрипта (сессия ssh-agent)

```bash
# Запускаем агент и добавляем ключ (спросит passphrase 1 раз)
eval "$(ssh-agent -s)"
ssh-add /root/.ssh/id_rsa_rsync

# Дальше rsync работает без пароля
./rsync_transfer.sh
```

Автоматизация через keychain (для cron)

```bash
apt-get install -y keychain

# В ~/.bashrc или в самом скрипте:
eval "$(keychain --eval --agents ssh id_rsa_rsync)"
```

Теперь passphrase вводится один раз, а дальше ключ «живёт» в агенте для всех последующих запусков, включая cron (через ~/.keychain/$(hostname)-sh).

---

Сравнение и что выбрать

Если сеть доверенная и хочется просто — берите sshpass + файл /root/.rsync_pass с правами 600. Минимум телодвижений, пароль не светится в ps.

Если безопасность важна — только SSH-ключ с passphrase + ssh-agent/keychain. Пароль пользователя на сервере вообще не используется, ключ зашифрован, passphrase вводится человеком или хранится в защищённом агенте.

Чего делать НЕ стоит:

· sshpass -p 'пароль' — пароль виден в ps aux любому пользователю системы
· хранить пароль в Git-репозитории, даже приватном
· отключать StrictHostKeyChecking=no глобально в ~/.ssh/config — это открывает MITM-атаку

Бонус: защита от утечки пароля через логи

sshpass при ошибке аутентификации может написать в stderr что-то вроде Permission denied, please try again — сам пароль не светит, но имя пользователя попадёт в лог. Если беспокоит — добавьте в скрипт фильтр:

```bash
# Перенаправляем rsync в лог, но чистим потенциально чувствительные строки
rsync ... 2>&1 | sed -E 's/backupuser@[0-9.]+/backupuser@<host>/g' >> "${LOG_FILE}"
```

И на всякий случай — не забудьте закрыть доступ к самому лог-каталогу:

```bash
chmod 700 /var/log/rsync_transfer
```

_____
_____


Исправление ошибки «код 127»

Код 127 в bash означает «command not found» — команда не найдена. Это не ошибка rsync (у него коды 0–35), а ошибка оболочки: rsync попытался запустить внешнюю команду из параметра -e, и не нашёл её.

Где именно возникает 127 в моём скрипте

Три типичные причины:

1. sshpass или ssh не в PATH (особенно в cron)

Когда скрипт запускается из cron или systemd, PATH=/usr/bin:/bin вместо обычного /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:.... Если sshpass лежит в /usr/local/bin или собран вручную — rsync его не найдёт и вернёт 127.

2. -e со строкой, где путь содержит пробелы или относительный путь

```bash
SSHPASS_CMD="sshpass -f ${PASS_FILE}"    # ← если PASS_FILE=/path with space/pass
-e "${SSHPASS_CMD} ssh ${SSH_OPTS}"      # ← rsync разобьёт по пробелам
```

rsync парсит -e по пробелам — путь с пробелом превратится в несуществующую команду → 127.

3. stat -c на системах, где его нет в PATH (редко, но бывает в минимальных образах)

---

Исправленный скрипт

Ключевые правки:

· явное разрешение абсолютных путей через command -v;
· нормализация PATH (важно для cron);
· использование абсолютных путей в -e, чтобы rsync не зависел от PATH;
· отдельная обработка кода 127 с понятным сообщением и выходом;
· проверка не только sshpass, но и ssh.

```bash
#!/bin/bash
#
# rsync_transfer_pass.sh — перенос с авторизацией по паролю
# Исправлена обработка кода 127 (command not found)
#

# ============ ГАРАНТИРОВАННЫЙ PATH ============
# Если запускается из cron — PATH может быть урезан.
# Добавляем стандартные каталоги, где обычно лежат ssh/sshpass.
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:${PATH}"

# ============ НАСТРОЙКИ ============
SOURCE_DIRS=(
    "/data/app1/"
    "/data/app2/"
)

REMOTE_USER="backupuser"
REMOTE_HOST="192.168.1.100"
REMOTE_PORT="22"
REMOTE_BASE_DIR="/backup/data"

PASS_FILE="/root/.rsync_pass"

LOG_DIR="/var/log/rsync_transfer"
LOG_FILE="${LOG_DIR}/rsync_$(date +%Y%m%d_%H%M%S).log"

MAX_RETRIES=3
SSH_TIMEOUT=30

# ============ ПОДГОТОВКА ============
mkdir -p "${LOG_DIR}"
umask 077

log() {
    local level="$1"; shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${level}] $*" | tee -a "${LOG_FILE}"
}

# ---- Разрешаем абсолютные пути к бинарникам ----
# Это ключевое исправление: rsync получает уже готовые полные пути
# и не зависит от PATH в момент вызова ssh.
SSHPASS_BIN="$(command -v sshpass 2>/dev/null)"
SSH_BIN="$(command -v ssh 2>/dev/null)"
RSYNC_BIN="$(command -v rsync 2>/dev/null)"

if [[ -z "${SSH_BIN}" ]]; then
    log "ERROR" "ssh не найден в PATH. Установите openssh-client."
    exit 1
fi

if [[ -z "${RSYNC_BIN}" ]]; then
    log "ERROR" "rsync не найден в PATH."
    exit 1
fi

if [[ -z "${SSHPASS_BIN}" ]]; then
    log "ERROR" "sshpass не найден. Установите: apt-get install sshpass"
    log "INFO"  "Пути поиска: ${PATH}"
    exit 1
fi

log "INFO" "Используемые бинарники: ssh=${SSH_BIN}, rsync=${RSYNC_BIN}, sshpass=${SSHPASS_BIN}"

# ---- Проверка файла с паролем ----
if [[ ! -f "${PASS_FILE}" ]]; then
    log "ERROR" "Файл с паролем не найден: ${PASS_FILE}"
    exit 1
fi

PERMS=$(stat -c '%a' "${PASS_FILE}" 2>/dev/null || echo "???")
if [[ "${PERMS}" != "600" && "${PERMS}" != "400" ]]; then
    log "WARN" "Права на ${PASS_FILE} = ${PERMS}. Рекомендуется 600."
fi

# ---- Формируем SSH-опции с АБСОЛЮТНЫМ путём к ssh ----
# Обратите внимание: заворачиваем в кавычки путь к ssh, если там пробелы.
SSH_OPTS="-p ${REMOTE_PORT} -o ConnectTimeout=${SSH_TIMEOUT} -o StrictHostKeyChecking=accept-new"

# Для sshpass: -f <file> <абсолютный_путь_ssh> <опции>
SSHPASS_CMD="${SSHPASS_BIN} -f ${PASS_FILE} ${SSH_BIN} ${SSH_OPTS}"

# ============ СТАРТ ============
log "INFO" "======================================================"
log "INFO" "Запуск переноса с авторизацией по паролю"
log "INFO" "Источник: ${SOURCE_DIRS[*]}"
log "INFO" "Назначение: ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BASE_DIR}"
log "INFO" "======================================================"

# Проверка соединения
log "INFO" "Проверка соединения с ${REMOTE_HOST}..."
if ! ${SSHPASS_BIN} -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
        "${REMOTE_USER}@${REMOTE_HOST}" "echo ok" &>>"${LOG_FILE}"; then
    log "ERROR" "Не удалось подключиться к ${REMOTE_HOST}. Проверьте пароль."
    exit 1
fi
log "INFO" "Соединение установлено."

# Создаём базовый каталог
${SSHPASS_BIN} -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
    "${REMOTE_USER}@${REMOTE_HOST}" "mkdir -p '${REMOTE_BASE_DIR}'" &>>"${LOG_FILE}" \
    || { log "ERROR" "Не удалось создать ${REMOTE_BASE_DIR}"; exit 1; }

# ============ ПЕРЕНОС ============
TOTAL=0; SUCCESS=0; FAILED=0
FAILED_DIRS=()

for SRC in "${SOURCE_DIRS[@]}"; do
    TOTAL=$((TOTAL + 1))

    if [[ ! -d "${SRC}" ]]; then
        log "WARN" "Источник не найден, пропуск: ${SRC}"
        FAILED=$((FAILED + 1))
        FAILED_DIRS+=("${SRC} (не существует)")
        continue
    fi

    DEST="${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BASE_DIR}/$(basename "${SRC}")"
    log "INFO" "------------------------------------------------------"
    log "INFO" "Перенос: ${SRC}  ->  ${DEST}"

    RETRY=0; RC=1
    while [[ ${RETRY} -lt ${MAX_RETRIES} && ${RC} -ne 0 ]]; do
        RETRY=$((RETRY + 1))
        log "INFO" "Попытка ${RETRY}/${MAX_RETRIES}..."

        # -e получает ГОТОВУЮ команду с абсолютными путями.
        # rsync запускает её через shell → 127 не должно быть.
        "${RSYNC_BIN}" -avz --delete \
            --numeric-ids \
            --partial --progress --stats --human-readable \
            -e "${SSHPASS_CMD}" \
            "${SRC}" "${DEST}" &>>"${LOG_FILE}"
        RC=$?

        case ${RC} in
            0)
                log "INFO" "Успешно: ${SRC}"
                SUCCESS=$((SUCCESS + 1))
                ;;
            127)
                # command not found — обычно sshpass/ssh не в PATH
                log "ERROR" "Код 127: команда не найдена. Проверьте, что sshpass и ssh доступны."
                log "ERROR" "PATH=${PATH}"
                log "ERROR" "ssh=${SSH_BIN}, sshpass=${SSHPASS_BIN}, rsync=${RSYNC_BIN}"
                # Повторять бессмысленно — 127 не исчезнет само
                FAILED=$((FAILED + 1))
                FAILED_DIRS+=("${SRC} (code 127)")
                RC=0   # выходим из цикла повторов
                break
                ;;
            *)
                log "WARN" "Ошибка rsync (код ${RC}) для ${SRC}"
                sleep 5
                ;;
        esac
    done
done

# ============ ИТОГИ ============
log "INFO" "======================================================"
log "INFO" "ИТОГИ: всего=${TOTAL}, успешно=${SUCCESS}, ошибок=${FAILED}"
for D in "${FAILED_DIRS[@]}"; do
    log "ERROR" "  - Не перенесено: ${D}"
done
log "INFO" "Лог сохранён: ${LOG_FILE}"

[[ ${FAILED} -eq 0 ]] && exit 0 || exit 2
```

---

Что именно изменилось (и почему это лечит 127)

Было Стало Зачем
-e "sshpass -f ... ssh ..." (голые имена) -e "${SSHPASS_BIN} -f ${PASS_FILE} ${SSH_BIN} ..." (абсолютные пути) rsync не зависит от PATH в момент вызова
Не трогал PATH export PATH="...стандартные пути...:${PATH}" чинит запуск из cron/systemd
Не проверял ssh и rsync command -v ssh, command -v rsync падаем с понятным сообщением, а не с 127
Код 127 попадал в общий *) и повторялся Отдельный case 127) → break не тратим 3 попытки на безнадёжную ошибку
Не логировались пути бинарников Логируем ssh=…, sshpass=…, rsync=… мгновенно видно, если что-то не разрешилось

---

Быстрая диагностика, если 127 всё равно остался

Выполните на сервере, где запускается скрипт:

```bash
# 1. Где бинарники?
which sshpass; which ssh; which rsync
#   должно вернуть: /usr/bin/sshpass, /usr/bin/ssh, /usr/bin/rsync (или /usr/local/...)

# 2. Что видит cron? Добавьте в crontab временно:
* * * * * echo "PATH=$PATH" >> /tmp/cron_path.log
#   и посмотрите: обычно это /usr/bin:/bin — без /usr/local/bin

# 3. Ручная проверка цепочки
/usr/bin/sshpass -f /root/.rsync_pass /usr/bin/ssh -p 22 user@host echo ok

# 4. Если sshpass установлен, но лежит нестандартно (например, /opt/bin)
#    — просто добавьте его каталог в начало PATH в скрипте:
export PATH="/opt/bin:${PATH}"
```

---

Альтернатива без sshpass (если 127 связан именно с ним)

Если sshpass в вашей системе в принципе отсутствует или запрещён политикой ИБ — используйте штатный механизм SSH через ssh-agent, тогда никаких внешних команд в -e не нужно и 127 исчезнет как класс:

```bash
# Один раз:
ssh-keygen -t ed25519 -f /root/.ssh/id_rsa_rsync -N ''   # без passphrase — если политика позволяет
ssh-copy-id -i /root/.ssh/id_rsa_rsync.pub user@host

# В скрипте вместо sshpass:
-e "ssh -i /root/.ssh/id_rsa_rsync -p ${REMOTE_PORT} -o ConnectTimeout=30"
```

Это, кстати, и правильнее с точки зрения безопасности — пароль пользователя на сервере вообще не участвует.

____________
____________
_____________
____________

В предоставленном скрипте есть несколько критических ошибок и логических недочетов, которые могут привести к сбою работы или некорректному поведению.

Ниже приведен подробный разбор ошибок от наиболее критичных к менее важным.

### 1. Критическая ошибка: Неправильная передача аргументов в `sshpass` (Цитирование)

**Проблема:**
Строка формирования команды:
```bash
SSHPASS_CMD="${SSHPASS_BIN} -f ${PASS_FILE} ${SSH_BIN} ${SSH_OPTS}"
```
И последующее использование:
```bash
-e "${SSHPASS_CMD}"
```

Когда вы заключаете переменную `${SSHPASS_CMD}` в двойные кавычки внутри вызова `rsync`, оболочка передает **всю строку целиком как один аргумент**.
То есть `rsync` попытается выполнить команду, где первым аргументом для shell будет не `sshpass`, а вся строка `"sshpass -f /root/.rsync_pass /usr/bin/ssh -p 22 ..."`. Shell не найдет исполняемый файл с таким длинным именем и вернет ошибку, либо `sshpass` не сможет корректно распарсить свои аргументы, так как они "склеились".

Кроме того, если в путях (`PASS_FILE`, `SSH_BIN`) или опциях (`SSH_OPTS`) есть пробелы, это сломает парсинг еще сильнее, даже без кавычек вокруг переменной.

**Решение:**
Не используйте одну строковую переменную для сложной команды с аргументами. Лучше использовать массив или функцию. Но самый надежный способ для `rsync -e` — передать готовую строку оболочки, но правильно экранированную, либо использовать массив Bash.

Однако, `rsync` ожидает, что после `-e` идет команда, которую он передаст в `/bin/sh -c`. Поэтому правильнее сформировать строку так, чтобы она была валидной командой для shell, и передать её без лишних оберток, либо использовать массив.

**Исправленный вариант (через массив, самый надежный):**
```bash
# Объявляем массив
RSYNC_SSH_CMD=( "${SSHPASS_BIN}" "-f" "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} )

# В вызове rsync используем раскрытие массива
# Обратите внимание: rsync -e ожидает строку. 
# Если передавать массив напрямую в -e, это может не сработать во всех версиях rsync.
# Стандартный подход для rsync -e: передать строку.

# Правильный способ формирования строки для -e:
SSHPASS_CMD_STR="${SSHPASS_BIN} -f '${PASS_FILE}' ${SSH_BIN} ${SSH_OPTS}"
# И использовать её так (без дополнительных кавычек вокруг всей переменной внутри -e, если она уже содержит структуру):
# Но rsync сам оборачивает это в sh -c. 

# Самый рабочий вариант для bash-скрипта:
-e "${SSHPASS_BIN} -f '${PASS_FILE}' ${SSH_BIN} ${SSH_OPTS}"
```
*Примечание:* В вашем коде `${SSH_OPTS}` уже содержит пробелы. Если вы делаете `"${SSHPASS_CMD}"`, то `rsync` видит один аргумент. Если вы уберете кавычки `$SSHPASS_CMD`, то слова разобьются по пробелам, но `rsync -e` принимает только **один** аргумент (команду).

**Рекомендация:** Используйте явное построение строки для `-e`:
```bash
RSYNC_SSH_STRING="${SSHPASS_BIN} -f '${PASS_FILE}' ${SSH_BIN} ${SSH_OPTS}"
# ...
-e "${RSYNC_SSH_STRING}"
```
*(Здесь важно экранировать путь к файлу пароля одинарными кавычками внутри строки, если там нет спецсимволов, чтобы shell внутри rsync его правильно прочитал).*

### 2. Ошибка безопасности и логики: Права доступа к файлу пароля

**Проблема:**
```bash
if [[ "${PERMS}" != "600" && "${PERMS}" != "400" ]]; then
    log "WARN" "Права на ${PASS_FILE} = ${PERMS}. Рекомендуется 600."
fi
```
Скрипт только предупреждает (`WARN`), но продолжает работу. Это плохо с точки зрения безопасности, но технически работает. Однако, если файл доступен для чтения другим пользователям, `sshpass` может отказать в работе (зависит от версии и настроек), или злоумышленник может прочитать пароль.

**Решение:**
Лучше принудительно выставить права или прервать выполнение, если права слишком открыты (например, 777).
```bash
chmod 600 "${PASS_FILE}" # Или exit 1, если политика строгая
```

### 3. Логическая ошибка: Обработка кода возврата 127 в цикле retry

**Проблема:**
```bash
case ${RC} in
    127)
        # ...
        RC=0   # выходим из цикла повторов
        break
        ;;
esac
```
Вы устанавливаете `RC=0` и делаете `break`. Цикл `while [[ ... RC -ne 0 ]]` действительно завершится.
Но далее код проверяет успех неявно. Переменная `SUCCESS` увеличивается только в ветке `0)`.
В ветке `127)` вы увеличиваете `FAILED`.
Это логически верно для счетчиков, но сбивает с толку чтение кода.

Более важная проблема: Код 127 при использовании `-e` часто возникает именно из-за ошибки №1 (неправильное цитирование). Исправив пункт 1, эта ветка, скорее всего, никогда не сработает, если бинарники найдены.

### 4. Проблема с `basename` и путями

**Проблема:**
```bash
DEST="${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BASE_DIR}/$(basename "${SRC}")"
```
Если `SOURCE_DIRS` содержит пути с trailing slash (как у вас: `/data/app1/`), `basename "/data/app1/"` вернет `app1`.
Это нормально. Но если вдруг в массиве окажется путь `/data/`, `basename` вернет `data`.
Убедитесь, что это ожидаемое поведение. Обычно это ок.

### 5. Отсутствие обработки специальных символов в путях

Если в `REMOTE_BASE_DIR` или именах папок есть пробелы или спецсимволы, конструкция:
```bash
"${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BASE_DIR}/$(basename "${SRC}")"
```
может сломаться, так как `rsync` передает эту строку удаленному хосту через SSH. Хотя `rsync` обычно экранирует аргументы, лучше быть осторожным.

В команде создания директории:
```bash
"mkdir -p '${REMOTE_BASE_DIR}'"
```
Использование одинарных кавычек внутри двойных защищает от пробелов в `REMOTE_BASE_DIR`. Это сделано правильно.

### 6. Мелкая ошибка: `umask` влияет на создаваемые файлы

Вы ставите `umask 077`.
Это означает, что все создаваемые файлы (включая лог-файл) будут иметь права `600` (для файлов) или `700` (для директорий).
Для лог-файла это хорошо.
Но убедитесь, что пользователь, под которым запускается скрипт (root), имеет право читать эти логи позже, если вы используете другого пользователя для мониторинга. Для root это не проблема.

### Исправленный фрагмент ключевой части (Rsync call)

Вот как нужно исправить формирование команды и вызов `rsync`, чтобы избежать ошибки 127 и проблем с аргументами:

```bash
# ... (предыдущий код без изменений до раздела ПЕРЕНОС) ...

# ============ ПЕРЕНОС ============

# Формируем строку команды для ssh, которую поймет remote shell.
# Важно: sshpass должен видеть флаг -f и файл.
# Мы экранируем путь к файлу пароля одинарными кавычками для безопасности пробелов.
RSYNC_RSH_CMD="${SSHPASS_BIN} -f '${PASS_FILE}' ${SSH_BIN} ${SSH_OPTS}"

TOTAL=0; SUCCESS=0; FAILED=0
FAILED_DIRS=()

for SRC in "${SOURCE_DIRS[@]}"; do
    TOTAL=$((TOTAL + 1))

    if [[ ! -d "${SRC}" ]]; then
        log "WARN" "Источник не найден, пропуск: ${SRC}"
        FAILED=$((FAILED + 1))
        FAILED_DIRS+=("${SRC} (не существует)")
        continue
    fi

    # Убираем trailing slash для basename, если он есть, хотя basename справляется
    DEST_PATH="${REMOTE_BASE_DIR}/$(basename "${SRC}")"
    DEST="${REMOTE_USER}@${REMOTE_HOST}:${DEST_PATH}"
    
    log "INFO" "------------------------------------------------------"
    log "INFO" "Перенос: ${SRC}  ->  ${DEST}"

    RETRY=0; RC=1
    while [[ ${RETRY} -lt ${MAX_RETRIES} && ${RC} -ne 0 ]]; do
        RETRY=$((RETRY + 1))
        log "INFO" "Попытка ${RETRY}/${MAX_RETRIES}..."

        # ВЫЗОВ RSYNC
        # Используем двойные кавычки вокруг всей строки RSYNC_RSH_CMD, 
        # так как rsync -e ожидает один аргумент-строку.
        "${RSYNC_BIN}" -avz --delete \
            --numeric-ids \
            --partial --progress --stats --human-readable \
            -e "${RSYNC_RSH_CMD}" \
            "${SRC}" "${DEST}" &>>"${LOG_FILE}"
        
        RC=$?

        case ${RC} in
            0)
                log "INFO" "Успешно: ${SRC}"
                SUCCESS=$((SUCCESS + 1))
                ;;
            127)
                log "ERROR" "Код 127: Команда не найдена. Проверьте пути к sshpass/ssh."
                log "ERROR" "Команда RSH: ${RSYNC_RSH_CMD}"
                FAILED=$((FAILED + 1))
                FAILED_DIRS+=("${SRC} (code 127)")
                break # Прерываем цикл retry, так как ошибка фатальная для конфигурации
                ;;
            *)
                log "WARN" "Ошибка rsync (код ${RC}) для ${SRC}. Повтор..."
                sleep 5
                ;;
        esac
    done
    
    # Если вышли из цикла по ошибке (не 0), убеждаемся, что FAILED учтен, если еще не
    if [[ ${RC} -ne 0 ]]; then
        # Проверка, не добавили ли мы уже этот каталог в FAILED_DIRS в случае 127
        # В случае других ошибок после исчерпания попыток:
        if [[ ${RC} -ne 127 ]]; then
             FAILED=$((FAILED + 1))
             FAILED_DIRS+=("${SRC} (failed after retries)")
        fi
    fi
done
```

### Резюме исправлений:

1.  **Главное:** Замените сборку команды через переменную `SSHPASS_CMD` и её использование в `-e` на правильное формирование строки `RSYNC_RSH_CMD` и передачу её в `-e` в кавычках.
2.  **Улучшение:** Добавьте проверку на существование директории назначения перед созданием (хотя `mkdir -p` это делает, но логика создания через отдельный SSH-вызов остается).
3.  **Логика:** Уточните обработку выхода из цикла `retry` для ошибок, отличных от 127, чтобы гарантированно инкрементировать счетчик `FAILED` после исчерпания попыток.

Эти изменения устранят ошибку "command not found" (127), которая является следствием неправильного парсинга аргументов `sshpass`.

_____
_____
_____
_____
_____


Вот полностью исправленный и готовый к использованию скрипт. 

В него внесены все критические исправления:
1. **Главное исправление:** Правильное формирование строки для `rsync -e`. Теперь пути и опции корректно передаются через локальный `sh -c`, что полностью исключает ошибку `127 (command not found)`.
2. **Исправление логики повторов (retry):** Если `rsync` завершился с ошибкой (не 0 и не 127) после всех попыток, каталог теперь гарантированно добавляется в список `FAILED_DIRS` и увеличивает счетчик ошибок (в оригинале при исчерпании попыток счетчик ошибок не рос).
3. **Безопасность путей:** Путь к файлу пароля внутри команды для `rsync` теперь экранируется двойными кавычками, что спасет от ошибок, если в пути случайно окажутся пробелы.

```bash
#!/bin/bash
#
# rsync_transfer_pass.sh — перенос с авторизацией по паролю
# Версия: 2.0 (Исправлены ошибки цитирования для rsync -e и логика retry)
#

# ============ ГАРАНТИРОВАННЫЙ PATH ============
# Если запускается из cron — PATH может быть урезан.
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:${PATH}"

# ============ НАСТРОЙКИ ============
SOURCE_DIRS=(
    "/data/app1/"
    "/data/app2/"
)

REMOTE_USER="backupuser"
REMOTE_HOST="192.168.1.100"
REMOTE_PORT="22"
REMOTE_BASE_DIR="/backup/data"

PASS_FILE="/root/.rsync_pass"

LOG_DIR="/var/log/rsync_transfer"
LOG_FILE="${LOG_DIR}/rsync_$(date +%Y%m%d_%H%M%S).log"

MAX_RETRIES=3
SSH_TIMEOUT=30

# ============ ПОДГОТОВКА ============
mkdir -p "${LOG_DIR}"
umask 077

log() {
    local level="$1"; shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${level}] $*" | tee -a "${LOG_FILE}"
}

# ---- Разрешаем абсолютные пути к бинарникам ----
SSHPASS_BIN="$(command -v sshpass 2>/dev/null)"
SSH_BIN="$(command -v ssh 2>/dev/null)"
RSYNC_BIN="$(command -v rsync 2>/dev/null)"

if [[ -z "${SSH_BIN}" ]]; then
    log "ERROR" "ssh не найден в PATH. Установите openssh-client."
    exit 1
fi

if [[ -z "${RSYNC_BIN}" ]]; then
    log "ERROR" "rsync не найден в PATH."
    exit 1
fi

if [[ -z "${SSHPASS_BIN}" ]]; then
    log "ERROR" "sshpass не найден. Установите: apt-get install sshpass"
    log "INFO"  "Пути поиска: ${PATH}"
    exit 1
fi

log "INFO" "Используемые бинарники: ssh=${SSH_BIN}, rsync=${RSYNC_BIN}, sshpass=${SSHPASS_BIN}"

# ---- Проверка файла с паролем ----
if [[ ! -f "${PASS_FILE}" ]]; then
    log "ERROR" "Файл с паролем не найден: ${PASS_FILE}"
    exit 1
fi

PERMS=$(stat -c '%a' "${PASS_FILE}" 2>/dev/null || echo "???")
if [[ "${PERMS}" != "600" && "${PERMS}" != "400" ]]; then
    log "WARN" "Права на ${PASS_FILE} = ${PERMS}. Рекомендуется 600 или 400."
    # Попытка автоматически исправить права, если мы root
    if [[ $EUID -eq 0 ]]; then
        chmod 600 "${PASS_FILE}" && log "INFO" "Права на ${PASS_FILE} автоматически изменены на 600."
    fi
fi

# ---- Формируем SSH-опции ----
SSH_OPTS="-p ${REMOTE_PORT} -o ConnectTimeout=${SSH_TIMEOUT} -o StrictHostKeyChecking=accept-new"

# ============ ИСПРАВЛЕНИЕ КРИТИЧЕСКОЙ ОШИБКИ ============
# Формируем строку для rsync -e. 
# rsync передает эту строку в локальный shell (sh -c). 
# Поэтому путь к файлу пароля оборачиваем в экранированные двойные кавычки \"...\", 
# чтобы защитить от пробелов в пути, а переменные раскрываются на этапе формирования строки.
RSYNC_RSH_CMD="${SSHPASS_BIN} -f \"${PASS_FILE}\" ${SSH_BIN} ${SSH_OPTS}"

# ============ СТАРТ ============
log "INFO" "======================================================"
log "INFO" "Запуск переноса с авторизацией по паролю"
log "INFO" "Источник: ${SOURCE_DIRS[*]}"
log "INFO" "Назначение: ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BASE_DIR}"
log "INFO" "======================================================"

# Проверка соединения (используем прямые переменные, а не RSYNC_RSH_CMD)
log "INFO" "Проверка соединения с ${REMOTE_HOST}..."
if ! "${SSHPASS_BIN}" -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
        "${REMOTE_USER}@${REMOTE_HOST}" "echo ok" &>>"${LOG_FILE}"; then
    log "ERROR" "Не удалось подключиться к ${REMOTE_HOST}. Проверьте пароль и доступность хоста."
    exit 1
fi
log "INFO" "Соединение установлено."

# Создаём базовый каталог
"${SSHPASS_BIN}" -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
    "${REMOTE_USER}@${REMOTE_HOST}" "mkdir -p '${REMOTE_BASE_DIR}'" &>>"${LOG_FILE}" \
    || { log "ERROR" "Не удалось создать ${REMOTE_BASE_DIR} на удаленном хосте"; exit 1; }

# ============ ПЕРЕНОС ============
TOTAL=0; SUCCESS=0; FAILED=0
FAILED_DIRS=()

for SRC in "${SOURCE_DIRS[@]}"; do
    TOTAL=$((TOTAL + 1))

    if [[ ! -d "${SRC}" ]]; then
        log "WARN" "Источник не найден, пропуск: ${SRC}"
        FAILED=$((FAILED + 1))
        FAILED_DIRS+=("${SRC} (не существует)")
        continue
    fi

    DEST="${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BASE_DIR}/$(basename "${SRC}")"
    log "INFO" "------------------------------------------------------"
    log "INFO" "Перенос: ${SRC}  ->  ${DEST}"

    RETRY=0; RC=1
    while [[ ${RETRY} -lt ${MAX_RETRIES} && ${RC} -ne 0 ]]; do
        RETRY=$((RETRY + 1))
        log "INFO" "Попытка ${RETRY}/${MAX_RETRIES}..."

        # Вызов rsync с правильно сформированной строкой для -e
        "${RSYNC_BIN}" -avz --delete \
            --numeric-ids \
            --partial --progress --stats --human-readable \
            -e "${RSYNC_RSH_CMD}" \
            "${SRC}" "${DEST}" &>>"${LOG_FILE}"
        RC=$?

        case ${RC} in
            0)
                log "INFO" "Успешно: ${SRC}"
                SUCCESS=$((SUCCESS + 1))
                ;;
            127)
                log "ERROR" "Код 127: команда не найдена. Проверьте пути к sshpass/ssh."
                log "ERROR" "Сформированная команда RSH: ${RSYNC_RSH_CMD}"
                FAILED=$((FAILED + 1))
                FAILED_DIRS+=("${SRC} (code 127)")
                RC=0   # Искусственно обнуляем, чтобы выйти из цикла while без повторных попыток
                break
                ;;
            *)
                log "WARN" "Ошибка rsync (код ${RC}) для ${SRC}"
                sleep 5
                ;;
        esac
    done

    # ============ ИСПРАВЛЕНИЕ ЛОГИКИ RETRY ============
    # Если цикл завершился, но RC всё еще не 0 (и это не была обработка 127),
    # значит исчерпаны все попытки. Гарантированно считаем это как FAILED.
    if [[ ${RC} -ne 0 ]]; then
        FAILED=$((FAILED + 1))
        FAILED_DIRS+=("${SRC} (ошибка после ${MAX_RETRIES} попыток, код ${RC})")
    fi
done

# ============ ИТОГИ ============
log "INFO" "======================================================"
log "INFO" "ИТОГИ: всего=${TOTAL}, успешно=${SUCCESS}, ошибок=${FAILED}"
if [[ ${#FAILED_DIRS[@]} -gt 0 ]]; then
    for D in "${FAILED_DIRS[@]}"; do
        log "ERROR" "  - Не перенесено: ${D}"
    done
fi
log "INFO" "Лог сохранён: ${LOG_FILE}"

# Возвращаем 0 если всё успешно, иначе 2 (чтобы cron/мониторинг могли перехватить ошибку)
[[ ${FAILED} -eq 0 ]] && exit 0 || exit 2
```

### Что было изменено (кратко):
1. **`RSYNC_RSH_CMD`**: Строка для `-e` теперь формируется как `${SSHPASS_BIN} -f \"${PASS_FILE}\" ${SSH_BIN} ${SSH_OPTS}`. Экранированные кавычки `\"` гарантируют, что когда `rsync` передаст эту команду в `sh -c`, путь к паролю не "развалится" на части, если в нем вдруг появятся пробелы.
2. **Счетчик ошибок в цикле `while`**: Добавлен блок `if [[ ${RC} -ne 0 ]]; then` после цикла. В оригинальном скрипте, если `rsync` падал с кодом, например, `23` (ошибка частичной передачи) 3 раза подряд, цикл завершался, но переменная `FAILED` не увеличивалась, и скрипт мог ошибочно рапортовать об успехе или просто не включить каталог в итоговый отчет.
3. **Автоисправление прав**: Если скрипт запущен от `root` и права на файл пароля неверные, он теперь сам выполнит `chmod 600`, а не просто напишет предупреждение (это предотвратит блокировку со стороны `sshpass`, который в новых версиях отказывается работать с "открытыми" файлами паролей).

________
________

# Анализ скриншота и исправление скрипта

## Распознанный вывод скрипта

```
[2026-10-09 17:17:21] [INFO] Используемые бинарники: ssh=/usr/bin/ssh, rsync=/usr/bin/rsync, sshpass=/...
[2026-10-09 17:17:21] [INFO] ======================================================
[2026-10-09 17:17:21] [INFO] Запуск переноса с авторизацией по паролю
[2026-10-09 17:17:21] [INFO] Источник: /tmp/app01
[2026-10-09 17:17:21] [INFO] Назначение: srvansible@nn-1cfls-001-p:/tmp/bk_app_all
[2026-10-09 17:17:21] [INFO] ======================================================
[2026-10-09 17:17:21] [INFO] Проверка соединения с nn-1cfls-001-p...
ok
[2026-10-09 17:17:22] [INFO] Соединение установлено.
[2026-10-09 17:17:22] [INFO] ------------------------------------------------------
[2026-10-09 17:17:22] [INFO] Перенос: /tmp/app01  ->  srvansible@nn-1cfls-001-p:/tmp/bk_app_all/app01
[2026-10-09 17:17:22] [INFO] Попытка 1/3...
bash: line 1: rsync: command not found
rsync: connection unexpectedly closed (0 bytes received so far) [sender]
rsync error: remote command not found (code 127) at io.c(232) [sender=3.2.7]
[2026-10-09 17:17:23] [ERROR] Код 127: команда не найдена. Проверьте пути к sshpass/ssh.
[2026-10-09 17:17:23] [ERROR] Сформированная команда RSH: /usr/bin/sshpass -f "/root/.rsync_pass" /usr/bin/ssh...
[2026-10-09 17:17:23] [INFO] ======================================================
[2026-10-09 17:17:23] [INFO] ИТОГИ: всего=1, успешно=0, ошибок=1
[2026-10-09 17:17:23] [ERROR]   - Не перенесено: /tmp/app01 (code 127)
[2026-10-09 17:17:23] [INFO] Лог сохранён: /var/log/rsync_transfer/rsync_20261009_171721.log
```

## Диагноз проблемы

**Ключевая строка:** `bash: line 1: rsync: command not found`

Проблема **НЕ в локальном rsync** (он работает, иначе бы не дошло до этой ошибки). Проблема в том, что **rsync не найден на УДАЛЁННОМ хосте** `nn-1cfls-001-p`.

Когда rsync работает по SSH, он:
1. Запускается локально
2. Через SSH подключается к удалённому хосту
3. **Пытается запустить rsync на удалённой стороне**
4. Если на удалённом хосте rsync не в PATH (или не установлен) → code 127

При использовании `sshpass` + non-interactive shell PATH часто урезан, и даже если rsync установлен в `/usr/bin/rsync`, shell его не находит.

## Исправленный скрипт

```bash
#!/bin/bash
#
# rsync_transfer_pass.sh — перенос с авторизацией по паролю
# Версия: 3.0 (Исправлена ошибка 127 на удалённом хосте)
#

# ============ ГАРАНТИРОВАННЫЙ PATH ============
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:${PATH}"

# ============ НАСТРОЙКИ ============
SOURCE_DIRS=(
    "/tmp/app01"
)

REMOTE_USER="srvansible"
REMOTE_HOST="nn-1cfls-001-p"
REMOTE_PORT="22"
REMOTE_BASE_DIR="/tmp/bk_app_all"

PASS_FILE="/root/.rsync_pass"

# ВАЖНО: Явный путь к rsync на удалённом хосте
# Если rsync установлен в нестандартном месте — измените этот путь
REMOTE_RSYNC_PATH="/usr/bin/rsync"

LOG_DIR="/var/log/rsync_transfer"
LOG_FILE="${LOG_DIR}/rsync_$(date +%Y%m%d_%H%M%S).log"

MAX_RETRIES=3
SSH_TIMEOUT=30

# ============ ПОДГОТОВКА ============
mkdir -p "${LOG_DIR}"
umask 077

log() {
    local level="$1"; shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${level}] $*" | tee -a "${LOG_FILE}"
}

# ---- Поиск бинарников ----
SSHPASS_BIN="$(command -v sshpass 2>/dev/null)"
SSH_BIN="$(command -v ssh 2>/dev/null)"
RSYNC_BIN="$(command -v rsync 2>/dev/null)"

if [[ -z "${SSH_BIN}" ]]; then
    log "ERROR" "ssh не найден в PATH. Установите openssh-client."
    exit 1
fi

if [[ -z "${RSYNC_BIN}" ]]; then
    log "ERROR" "rsync не найден в PATH локально."
    exit 1
fi

if [[ -z "${SSHPASS_BIN}" ]]; then
    log "ERROR" "sshpass не найден. Установите: apt-get install sshpass"
    exit 1
fi

log "INFO" "Локальные бинарники: ssh=${SSH_BIN}, rsync=${RSYNC_BIN}, sshpass=${SSHPASS_BIN}"
log "INFO" "Удалённый rsync путь: ${REMOTE_RSYNC_PATH}"

# ---- Проверка файла с паролем ----
if [[ ! -f "${PASS_FILE}" ]]; then
    log "ERROR" "Файл с паролем не найден: ${PASS_FILE}"
    exit 1
fi

PERMS=$(stat -c '%a' "${PASS_FILE}" 2>/dev/null || echo "???")
if [[ "${PERMS}" != "600" && "${PERMS}" != "400" ]]; then
    log "WARN" "Права на ${PASS_FILE} = ${PERMS}. Исправляем на 600."
    if [[ $EUID -eq 0 ]]; then
        chmod 600 "${PASS_FILE}"
    fi
fi

# ---- SSH-опции ----
SSH_OPTS="-p ${REMOTE_PORT} -o ConnectTimeout=${SSH_TIMEOUT} -o StrictHostKeyChecking=accept-new"

# ---- Команда для rsync -e ----
# Экранированные кавычки вокруг пути к паролю
RSYNC_RSH_CMD="${SSHPASS_BIN} -f \"${PASS_FILE}\" ${SSH_BIN} ${SSH_OPTS}"

# ============ СТАРТ ============
log "INFO" "======================================================"
log "INFO" "Запуск переноса с авторизацией по паролю"
log "INFO" "Источник: ${SOURCE_DIRS[*]}"
log "INFO" "Назначение: ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BASE_DIR}"
log "INFO" "======================================================"

# Проверка соединения
log "INFO" "Проверка соединения с ${REMOTE_HOST}..."
if ! "${SSHPASS_BIN}" -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
        "${REMOTE_USER}@${REMOTE_HOST}" "echo ok" &>>"${LOG_FILE}"; then
    log "ERROR" "Не удалось подключиться к ${REMOTE_HOST}."
    exit 1
fi
log "INFO" "Соединение установлено."

# ============ НОВОЕ: Проверка rsync на удалённом хосте ============
log "INFO" "Проверка наличия rsync на удалённом хосте..."
REMOTE_RSYNC_CHECK=$("${SSHPASS_BIN}" -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
    "${REMOTE_USER}@${REMOTE_HOST}" "which rsync 2>/dev/null || echo NOT_FOUND" 2>/dev/null)

if [[ "${REMOTE_RSYNC_CHECK}" == "NOT_FOUND" || -z "${REMOTE_RSYNC_CHECK}" ]]; then
    log "ERROR" "rsync НЕ НАЙДЕН на удалённом хосте ${REMOTE_HOST}!"
    log "ERROR" "Установите rsync на удалённом хосте:"
    log "ERROR" "  sudo apt-get install rsync  (Debian/Ubuntu)"
    log "ERROR" "  sudo yum install rsync      (CentOS/RHEL)"
    exit 1
fi

log "INFO" "rsync на удалённом хосте найден: ${REMOTE_RSYNC_CHECK}"

# Если путь не совпадает с заданным — используем найденный
if [[ "${REMOTE_RSYNC_CHECK}" != "${REMOTE_RSYNC_PATH}" ]]; then
    log "WARN" "Путь к rsync отличается от заданного. Используем: ${REMOTE_RSYNC_CHECK}"
    REMOTE_RSYNC_PATH="${REMOTE_RSYNC_CHECK}"
fi

# Создаём базовый каталог
"${SSHPASS_BIN}" -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
    "${REMOTE_USER}@${REMOTE_HOST}" "mkdir -p '${REMOTE_BASE_DIR}'" &>>"${LOG_FILE}" \
    || { log "ERROR" "Не удалось создать ${REMOTE_BASE_DIR}"; exit 1; }

# ============ ПЕРЕНОС ============
TOTAL=0; SUCCESS=0; FAILED=0
FAILED_DIRS=()

for SRC in "${SOURCE_DIRS[@]}"; do
    TOTAL=$((TOTAL + 1))

    if [[ ! -d "${SRC}" ]]; then
        log "WARN" "Источник не найден, пропуск: ${SRC}"
        FAILED=$((FAILED + 1))
        FAILED_DIRS+=("${SRC} (не существует)")
        continue
    fi

    DEST="${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BASE_DIR}/$(basename "${SRC}")"
    log "INFO" "------------------------------------------------------"
    log "INFO" "Перенос: ${SRC}  ->  ${DEST}"

    RETRY=0; RC=1
    while [[ ${RETRY} -lt ${MAX_RETRIES} && ${RC} -ne 0 ]]; do
        RETRY=$((RETRY + 1))
        log "INFO" "Попытка ${RETRY}/${MAX_RETRIES}..."

        # КЛЮЧЕВОЕ ИСПРАВЛЕНИЕ: --rsync-path указывает явный путь к rsync на удалённом хосте
        "${RSYNC_BIN}" -avz --delete \
            --numeric-ids \
            --partial --progress --stats --human-readable \
            --rsync-path="${REMOTE_RSYNC_PATH}" \
            -e "${RSYNC_RSH_CMD}" \
            "${SRC}" "${DEST}" &>>"${LOG_FILE}"
        RC=$?

        case ${RC} in
            0)
                log "INFO" "Успешно: ${SRC}"
                SUCCESS=$((SUCCESS + 1))
                ;;
            127)
                log "ERROR" "Код 127: команда не найдена на удалённом хосте."
                log "ERROR" "Проверьте, что rsync установлен на ${REMOTE_HOST}"
                log "ERROR" "Используемый путь: ${REMOTE_RSYNC_PATH}"
                FAILED=$((FAILED + 1))
                FAILED_DIRS+=("${SRC} (code 127)")
                RC=0
                break
                ;;
            *)
                log "WARN" "Ошибка rsync (код ${RC}) для ${SRC}"
                sleep 5
                ;;
        esac
    done

    if [[ ${RC} -ne 0 ]]; then
        FAILED=$((FAILED + 1))
        FAILED_DIRS+=("${SRC} (ошибка после ${MAX_RETRIES} попыток, код ${RC})")
    fi
done

# ============ ИТОГИ ============
log "INFO" "======================================================"
log "INFO" "ИТОГИ: всего=${TOTAL}, успешно=${SUCCESS}, ошибок=${FAILED}"
if [[ ${#FAILED_DIRS[@]} -gt 0 ]]; then
    for D in "${FAILED_DIRS[@]}"; do
        log "ERROR" "  - Не перенесено: ${D}"
    done
fi
log "INFO" "Лог сохранён: ${LOG_FILE}"

[[ ${FAILED} -eq 0 ]] && exit 0 || exit 2
```

## Что изменилось

| Изменение | Описание |
|-----------|----------|
| **`REMOTE_RSYNC_PATH`** | Новая переменная с явным путём к rsync на удалённом хосте (`/usr/bin/rsync`) |
| **`--rsync-path`** | Добавлен в вызов rsync — указывает, какой бинарник запускать на удалённой стороне |
| **Проверка удалённого rsync** | Перед переносом скрипт проверяет наличие rsync на удалённом хосте через `which rsync` |
| **Автоопределение пути** | Если `which rsync` находит другой путь — скрипт использует его |

## Быстрое решение без изменения скрипта

Если нужно срочно запустить перенос, можно просто добавить одну опцию в существующий вызов rsync:

```bash
--rsync-path="/usr/bin/rsync"
```

Или установить rsync на удалённом хосте:
```bash
ssh srvansible@nn-1cfls-001-p "sudo apt-get install -y rsync"
```