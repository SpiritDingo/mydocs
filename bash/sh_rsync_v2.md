# Исправленный скрипт с повышенными правами и отслеживанием изменений

```bash
#!/bin/bash
#
# rsync_transfer_pass.sh — перенос с авторизацией по паролю
# Версия: 5.0 (sudo локально и удалённо, отслеживание изменений)
#

# ============ ГАРАНТИРОВАННЫЙ PATH ============
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:${PATH}"

# ============ НАСТРОЙКИ ============
SOURCE_DIRS=(
    "/tmp/app01"
)

REMOTE_USER="user001"
REMOTE_HOST="srv001"
REMOTE_PORT="22"
REMOTE_BASE_DIR="/tmp/bk_app_all"

PASS_FILE="/root/.rsync_pass"

# Пути к rsync
LOCAL_RSYNC_PATH="/usr/bin/rsync"
REMOTE_RSYNC_PATH="/usr/bin/rsync"

# ============ НАСТРОЙКИ SUDO ============

# Локальный sudo: rsync запускается с правами root для чтения защищённых файлов
USE_LOCAL_SUDO="yes"

# Удалённый sudo: rsync на целевом хосте запускается с правами root для записи
USE_REMOTE_SUDO="yes"

# ============ ОТСЛЕЖИВАНИЕ ИЗМЕНЕНИЙ ============

# --checksum: сравнивать файлы по контрольной сумме, а не по времени/размеру
# Это гарантирует обнаружение изменений даже если timestamp не изменился
USE_CHECKSUM="yes"

# --itemize-changes: выводить список изменённых файлов
ITEMIZE_CHANGES="yes"

# --log-file: лог изменений rsync
RSYNC_CHANGE_LOG="${LOG_DIR}/changes_$(date +%Y%m%d_%H%M%S).log"

# ============ НАСТРОЙКИ ПРАВ ДОСТУПА ============

# Права на создаваемые файлы и директории (если не используется sudo)
RSYNC_CHMOD="Du=rwx,Dg=rx,Do=rx,Fu=rw,Fg=r,Fo=r"

# Umask на удалённой стороне (альтернатива --chmod)
REMOTE_UMASK="022"

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
log "INFO" "Локальный sudo: ${USE_LOCAL_SUDO}, Удалённый sudo: ${USE_REMOTE_SUDO}"

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

# ---- Команда для rsync -e (SSH через sshpass) ----
RSYNC_RSH_CMD="${SSHPASS_BIN} -f \"${PASS_FILE}\" ${SSH_BIN} ${SSH_OPTS}"

# ---- Формирование команд rsync с учётом sudo ----

# Локальная команда rsync
if [[ "${USE_LOCAL_SUDO}" == "yes" ]]; then
    LOCAL_RSYNC_CMD="sudo ${LOCAL_RSYNC_PATH}"
    log "INFO" "Локальный rsync будет запускаться через sudo"
else
    LOCAL_RSYNC_CMD="${LOCAL_RSYNC_PATH}"
fi

# Удалённая команда rsync
if [[ "${USE_REMOTE_SUDO}" == "yes" ]]; then
    REMOTE_RSYNC_FULL="sudo ${REMOTE_RSYNC_PATH}"
    log "INFO" "Удалённый rsync будет запускаться через sudo"
else
    REMOTE_RSYNC_FULL="${REMOTE_RSYNC_PATH}"
fi

# ============ ПРОВЕРКА SUDO ============

# Проверка локального sudo
if [[ "${USE_LOCAL_SUDO}" == "yes" ]]; then
    log "INFO" "Проверка локальных прав sudo..."
    if ! sudo -n ${LOCAL_RSYNC_PATH} --version &>/dev/null; then
        log "ERROR" "Не удалось выполнить rsync через локальный sudo!"
        log "ERROR" "Проверьте настройки sudoers или отключите USE_LOCAL_SUDO"
        exit 1
    fi
    log "INFO" "Локальный sudo для rsync работает корректно"
fi

# ============ СТАРТ ============
log "INFO" "======================================================"
log "INFO" "Запуск переноса с авторизацией по паролю"
log "INFO" "Источник: ${SOURCE_DIRS[*]}"
log "INFO" "Назначение: ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BASE_DIR}"
log "INFO" "Отслеживание изменений: checksum=${USE_CHECKSUM}, itemize=${ITEMIZE_CHANGES}"
log "INFO" "======================================================"

# Проверка соединения
log "INFO" "Проверка соединения с ${REMOTE_HOST}..."
if ! "${SSHPASS_BIN}" -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
        "${REMOTE_USER}@${REMOTE_HOST}" "echo ok" &>>"${LOG_FILE}"; then
    log "ERROR" "Не удалось подключиться к ${REMOTE_HOST}."
    exit 1
fi
log "INFO" "Соединение установлено."

# Проверка rsync на удалённом хосте
log "INFO" "Проверка наличия rsync на удалённом хосте..."
REMOTE_RSYNC_CHECK=$("${SSHPASS_BIN}" -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
    "${REMOTE_USER}@${REMOTE_HOST}" "which rsync 2>/dev/null || echo NOT_FOUND" 2>/dev/null)

if [[ "${REMOTE_RSYNC_CHECK}" == "NOT_FOUND" || -z "${REMOTE_RSYNC_CHECK}" ]]; then
    log "ERROR" "rsync НЕ НАЙДЕН на удалённом хосте ${REMOTE_HOST}!"
    log "ERROR" "Установите rsync: sudo apt-get install rsync"
    exit 1
fi

log "INFO" "rsync на удалённом хосте найден: ${REMOTE_RSYNC_CHECK}"

# Проверка удалённого sudo
if [[ "${USE_REMOTE_SUDO}" == "yes" ]]; then
    log "INFO" "Проверка прав sudo для rsync на удалённом хосте..."
    SUDO_CHECK=$("${SSHPASS_BIN}" -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
        "${REMOTE_USER}@${REMOTE_HOST}" "sudo -n ${REMOTE_RSYNC_PATH} --version 2>&1 | head -1" 2>/dev/null)
    
    if echo "${SUDO_CHECK}" | grep -q "rsync version"; then
        log "INFO" "sudo для rsync на удалённом хосте работает корректно"
    else
        log "ERROR" "Не удалось выполнить rsync через sudo на удалённом хосте!"
        log "ERROR" "Настройте sudoers на ${REMOTE_HOST}:"
        log "ERROR" "  ${REMOTE_USER} ALL=(root) NOPASSWD: ${REMOTE_RSYNC_PATH}"
        log "ERROR" "Или отключите USE_REMOTE_SUDO в скрипте"
        exit 1
    fi
fi

# Создаём базовый каталог на удалённом хосте
log "INFO" "Создание базового каталога ${REMOTE_BASE_DIR}..."
if [[ "${USE_REMOTE_SUDO}" == "yes" ]]; then
    "${SSHPASS_BIN}" -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
        "${REMOTE_USER}@${REMOTE_HOST}" "sudo mkdir -p '${REMOTE_BASE_DIR}'" &>>"${LOG_FILE}" \
        || { log "ERROR" "Не удалось создать ${REMOTE_BASE_DIR} (sudo)"; exit 1; }
else
    "${SSHPASS_BIN}" -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
        "${REMOTE_USER}@${REMOTE_HOST}" "mkdir -p '${REMOTE_BASE_DIR}'" &>>"${LOG_FILE}" \
        || { log "ERROR" "Не удалось создать ${REMOTE_BASE_DIR}"; exit 1; }
fi
log "INFO" "Базовый каталог создан."

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

    # Формируем опции для отслеживания изменений
    CHANGE_OPTS=""
    if [[ "${USE_CHECKSUM}" == "yes" ]]; then
        CHANGE_OPTS="${CHANGE_OPTS} --checksum"
        log "INFO" "Используется проверка по контрольной сумме (checksum)"
    fi
    
    if [[ "${ITEMIZE_CHANGES}" == "yes" ]]; then
        CHANGE_OPTS="${CHANGE_OPTS} --itemize-changes"
        log "INFO" "Включён вывод списка изменений (itemize-changes)"
    fi

    # Формируем опции прав доступа
    PERMS_OPTS=""
    if [[ "${USE_REMOTE_SUDO}" == "no" && -n "${RSYNC_CHMOD}" ]]; then
        PERMS_OPTS="--chmod=${RSYNC_CHMOD}"
    fi

    RETRY=0; RC=1
    while [[ ${RETRY} -lt ${MAX_RETRIES} && ${RC} -ne 0 ]]; do
        RETRY=$((RETRY + 1))
        log "INFO" "Попытка ${RETRY}/${MAX_RETRIES}..."

        # Запуск rsync с локальным sudo
        # sudo запускается локально, sshpass передаёт пароль для SSH
        sudo ${LOCAL_RSYNC_PATH} -avz --delete \
            --numeric-ids \
            --partial --progress --stats --human-readable \
            --rsync-path="${REMOTE_RSYNC_FULL}" \
            -e "${RSYNC_RSH_CMD}" \
            ${CHANGE_OPTS} \
            ${PERMS_OPTS} \
            --log-file="${RSYNC_CHANGE_LOG}" \
            "${SRC}" "${DEST}" &>>"${LOG_FILE}"
        
        RC=$?

        case ${RC} in
            0)
                log "INFO" "Успешно: ${SRC}"
                SUCCESS=$((SUCCESS + 1))
                
                # Выводим статистику изменений из лога
                if [[ -f "${RSYNC_CHANGE_LOG}" ]]; then
                    CHANGED_FILES=$(grep -c "^." "${RSYNC_CHANGE_LOG}" 2>/dev/null || echo "0")
                    log "INFO" "Изменений зафиксировано: ${CHANGED_FILES} (см. ${RSYNC_CHANGE_LOG})"
                fi
                ;;
            127)
                log "ERROR" "Код 127: команда не найдена."
                log "ERROR" "Локальная команда: ${LOCAL_RSYNC_CMD}"
                log "ERROR" "Удалённая команда: ${REMOTE_RSYNC_FULL}"
                if [[ "${USE_REMOTE_SUDO}" == "yes" ]]; then
                    log "ERROR" "Проверьте настройку sudoers на удалённом хосте"
                fi
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
log "INFO" "Основной лог: ${LOG_FILE}"
log "INFO" "Лог изменений: ${RSYNC_CHANGE_LOG}"
log "INFO" "======================================================"

[[ ${FAILED} -eq 0 ]] && exit 0 || exit 2
```

## Что изменилось

### 1. Пользователь и хост
- `REMOTE_USER="user001"` (было `srvansible`)
- `REMOTE_HOST="srv001"` (было `nn-1cfls-001-p`)

### 2. Повышение прав локально
```bash
USE_LOCAL_SUDO="yes"
```
Теперь rsync запускается через `sudo` на локальной машине, что позволяет читать файлы с правами root и других пользователей.

### 3. Повышение прав удалённо
```bash
USE_REMOTE_SUDO="yes"
```
На удалённом хосте rsync тоже запускается через sudo для записи в защищённые директории.

### 4. Отслеживание изменений
```bash
USE_CHECKSUM="yes"          # Сравнение по контрольной сумме
ITEMIZE_CHANGES="yes"       # Вывод списка изменённых файлов
RSYNC_CHANGE_LOG="..."      # Отдельный лог изменений
```

### 5. Настройка sudoers

**На локальном хосте:**
```bash
# Если скрипт запускается не от root, добавьте в /etc/sudoers:
youruser ALL=(root) NOPASSWD: /usr/bin/rsync
```

**На удалённом хосте (srv001):**
```bash
# Выполните на srv001:
sudo visudo

# Добавьте строку:
user001 ALL=(root) NOPASSWD: /usr/bin/rsync
```

### 6. Запуск скрипта

```bash
# От root (рекомендуется):
sudo ./rsync_transfer_pass.sh

# Или от обычного пользователя с настроенным sudo:
./rsync_transfer_pass.sh
```

### 7. Логи

- **Основной лог**: `/var/log/rsync_transfer/rsync_YYYYMMDD_HHMMSS.log`
- **Лог изменений**: `/var/log/rsync_transfer/changes_YYYYMMDD_HHMMSS.log`

Лог изменений содержит список всех переданных файлов с отметками:
- `>f.st....` — файл отправлен (send)
- `*deleting` — файл удалён на приёмнике
- `>f++++++++` — новый файл