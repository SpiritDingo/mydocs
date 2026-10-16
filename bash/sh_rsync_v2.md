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
- `REMOTE_USER="user001"` (было `test1`)
- `REMOTE_HOST="srv001"` (было `co-test-p`)

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

_____


# Скрипт копирования файлов с отслеживанием изменений

```bash
#!/bin/bash
#
# rsync_transfer_pass.sh — копирование файлов с отслеживанием изменений
# Версия: 8.0 (Детальный отчёт об изменениях, статистика, история)
#

# ============ ГАРАНТИРОВАННЫЙ PATH ============
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:${PATH}"

# ============ НАСТРОЙКИ ============
SOURCE_DIRS=(
    "/lc_file_storage/"
)

REMOTE_USER="user001"
REMOTE_HOST="srv001"
REMOTE_PORT="22"
REMOTE_BASE_DIR="/lc_file_storage"

PASS_FILE="/root/.rsync_pass"

# Пути к бинарникам
LOCAL_RSYNC_PATH="/usr/bin/rsync"
REMOTE_RSYNC_PATH="/usr/bin/rsync"

# ============ НАСТРОЙКИ SUDO ============
USE_LOCAL_SUDO="no"
USE_REMOTE_SUDO="yes"

# ============ ОТСЛЕЖИВАНИЕ ИЗМЕНЕНИЙ ============
# Использовать контрольные суммы для точного обнаружения изменений
USE_CHECKSUM="yes"

# Детальный список изменений (itemize)
ITEMIZE_CHANGES="yes"

# Сохранять историю изменений между запусками
SAVE_CHANGE_HISTORY="yes"

# Директория для хранения истории
HISTORY_DIR="/var/log/rsync_transfer/history"

# ============ НАСТРОЙКИ ПРАВ ДОСТУПА ============
RSYNC_CHMOD="Du=rwx,Dg=rx,Do=rx,Fu=rw,Fg=r,Fo=r"
REMOTE_UMASK="022"

# ============ ЛОГИРОВАНИЕ ============
LOG_DIR="/var/log/rsync_transfer"
LOG_FILE="${LOG_DIR}/rsync_$(date +%Y%m%d_%H%M%S).log"
RSYNC_CHANGE_LOG="${LOG_DIR}/changes_$(date +%Y%m%d_%H%M%S).log"
CHANGE_REPORT="${LOG_DIR}/report_$(date +%Y%m%d_%H%M%S).txt"
STATS_FILE="${LOG_DIR}/stats_$(date +%Y%m%d_%H%M%S).txt"

MAX_RETRIES=3
SSH_TIMEOUT=30

# ============ ПОДГОТОВКА ============
mkdir -p "${LOG_DIR}" "${HISTORY_DIR}"
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
    log "ERROR" "ssh не найден в PATH."
    exit 1
fi

if [[ -z "${RSYNC_BIN}" ]]; then
    log "ERROR" "rsync не найден в PATH локально."
    exit 1
fi

if [[ -z "${SSHPASS_BIN}" ]]; then
    log "ERROR" "sshpass не найден."
    exit 1
fi

log "INFO" "Бинарники: ssh=${SSH_BIN}, rsync=${RSYNC_BIN}, sshpass=${SSHPASS_BIN}"
log "INFO" "Шифрование: отключено, Отслеживание изменений: включено"

# ---- Проверка файла с паролем ----
if [[ ! -f "${PASS_FILE}" ]]; then
    log "ERROR" "Файл с паролем не найден: ${PASS_FILE}"
    exit 1
fi

PERMS=$(stat -c '%a' "${PASS_FILE}" 2>/dev/null || echo "???")
if [[ "${PERMS}" != "600" && "${PERMS}" != "400" ]]; then
    log "WARN" "Права на ${PASS_FILE} = ${PERMS}. Исправляем на 600."
    [[ $EUID -eq 0 ]] && chmod 600 "${PASS_FILE}"
fi

# ---- SSH-опции ----
SSH_OPTS="-t -t -p ${REMOTE_PORT} -o ConnectTimeout=${SSH_TIMEOUT} -o StrictHostKeyChecking=accept-new"

# ---- Команда для rsync -e ----
RSYNC_RSH_CMD="${SSHPASS_BIN} -f \"${PASS_FILE}\" ${SSH_BIN} ${SSH_OPTS}"

# ---- Команды rsync с учётом sudo ----
if [[ "${USE_LOCAL_SUDO}" == "yes" && $EUID -ne 0 ]]; then
    LOCAL_RSYNC_CMD="sudo ${LOCAL_RSYNC_PATH}"
else
    LOCAL_RSYNC_CMD="${LOCAL_RSYNC_PATH}"
    [[ $EUID -eq 0 ]] && log "INFO" "Запуск от root, локальный sudo не требуется"
fi

if [[ "${USE_REMOTE_SUDO}" == "yes" ]]; then
    REMOTE_RSYNC_FULL="sudo ${REMOTE_RSYNC_PATH}"
else
    REMOTE_RSYNC_FULL="${REMOTE_RSYNC_PATH}"
fi

# ============ ФУНКЦИИ ОТСЛЕЖИВАНИЯ ============

# Функция для генерации отчёта об изменениях
generate_change_report() {
    local change_log="$1"
    local report_file="$2"
    
    if [[ ! -f "${change_log}" ]]; then
        echo "Лог изменений не найден: ${change_log}" > "${report_file}"
        return
    fi
    
    # Подсчёт статистики
    local new_files=$(grep -c "^>f\.\.\.\.\.\.\." "${change_log}" 2>/dev/null || echo "0")
    local updated_files=$(grep -c "^>f.*[a-z]" "${change_log}" 2>/dev/null | grep -v "^>f\.\.\.\.\.\.\." || echo "0")
    local deleted_files=$(grep -c "^\*deleting" "${change_log}" 2>/dev/null || echo "0")
    local new_dirs=$(grep -c "^>d\.\.\.\.\.\.\." "${change_log}" 2>/dev/null || echo "0")
    local total_changes=$((new_files + updated_files + deleted_files + new_dirs))
    
    # Генерация отчёта
    {
        echo "=================================================="
        echo "ОТЧЁТ ОБ ИЗМЕНЕНИЯХ"
        echo "Дата: $(date '+%Y-%m-%d %H:%M:%S')"
        echo "Источник: ${SOURCE_DIRS[*]}"
        echo "Назначение: ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BASE_DIR}"
        echo "=================================================="
        echo ""
        echo "СТАТИСТИКА:"
        echo "  Новых файлов:      ${new_files}"
        echo "  Обновлённых файлов: ${updated_files}"
        echo "  Удалённых файлов:  ${deleted_files}"
        echo "  Новых директорий:  ${new_dirs}"
        echo "  ВСЕГО изменений:   ${total_changes}"
        echo ""
        echo "=================================================="
        echo "ДЕТАЛЬНЫЙ СПИСОК ИЗМЕНЕНИЙ:"
        echo "=================================================="
        echo ""
        
        if [[ ${new_files} -gt 0 ]]; then
            echo "НОВЫЕ ФАЙЛЫ:"
            grep "^>f\.\.\.\.\.\.\." "${change_log}" | awk '{print "  + " $2}'
            echo ""
        fi
        
        if [[ ${updated_files} -gt 0 ]]; then
            echo "ОБНОВЛЁННЫЕ ФАЙЛЫ:"
            grep "^>f" "${change_log}" | grep -v "^>f\.\.\.\.\.\.\." | awk '{print "  ~ " $2}'
            echo ""
        fi
        
        if [[ ${deleted_files} -gt 0 ]]; then
            echo "УДАЛЁННЫЕ ФАЙЛЫ:"
            grep "^\*deleting" "${change_log}" | awk '{print "  - " $2}'
            echo ""
        fi
        
        if [[ ${new_dirs} -gt 0 ]]; then
            echo "НОВЫЕ ДИРЕКТОРИИ:"
            grep "^>d\.\.\.\.\.\.\." "${change_log}" | awk '{print "  + " $2 "/"}'
            echo ""
        fi
        
        echo "=================================================="
        echo "Коды изменений:"
        echo "  > - файл отправлен"
        echo "  * - файл удалён"
        echo "  f - файл, d - директория"
        echo "  . - без изменений, буква - тип изменения"
        echo "=================================================="
    } > "${report_file}"
    
    log "INFO" "Отчёт об изменениях сохранён: ${report_file}"
}

# Функция для сохранения истории изменений
save_change_history() {
    local change_log="$1"
    local history_file="${HISTORY_DIR}/history_$(date +%Y%m%d).log"
    
    if [[ -f "${change_log}" ]]; then
        echo "=== Запуск $(date '+%Y-%m-%d %H:%M:%S') ===" >> "${history_file}"
        cat "${change_log}" >> "${history_file}"
        echo "" >> "${history_file}"
        log "INFO" "История изменений обновлена: ${history_file}"
    fi
}

# Функция для получения статистики передачи
get_transfer_stats() {
    local change_log="$1"
    local stats_file="$2"
    
    if [[ ! -f "${change_log}" ]]; then
        return
    fi
    
    # Извлечение статистики из лога rsync
    {
        echo "СТАТИСТИКА ПЕРЕДАЧИ"
        echo "==================="
        grep -A 20 "Number of files:" "${change_log}" | head -20
    } > "${stats_file}"
    
    log "INFO" "Статистика передачи сохранена: ${stats_file}"
}

# ============ СТАРТ ============
log "INFO" "======================================================"
log "INFO" "Запуск копирования файлов с отслеживанием изменений"
log "INFO" "Источник: ${SOURCE_DIRS[*]}"
log "INFO" "Назначение: ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_BASE_DIR}"
log "INFO" "Отслеживание: checksum=${USE_CHECKSUM}, itemize=${ITEMIZE_CHANGES}"
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
        exit 1
    fi
fi

# Создаём базовый каталог на удалённом хосте
log "INFO" "Создание/проверка базового каталога ${REMOTE_BASE_DIR}..."
if [[ "${USE_REMOTE_SUDO}" == "yes" ]]; then
    MKDIR_RESULT=$("${SSHPASS_BIN}" -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
        "${REMOTE_USER}@${REMOTE_HOST}" "sudo mkdir -p '${REMOTE_BASE_DIR}' && sudo chmod 755 '${REMOTE_BASE_DIR}' && echo OK" 2>&1)
    if [[ "${MKDIR_RESULT}" != *"OK"* ]]; then
        log "ERROR" "Не удалось создать/проверить ${REMOTE_BASE_DIR} (sudo)"
        log "ERROR" "Результат: ${MKDIR_RESULT}"
        exit 1
    fi
else
    "${SSHPASS_BIN}" -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} \
        "${REMOTE_USER}@${REMOTE_HOST}" "mkdir -p '${REMOTE_BASE_DIR}'" &>>"${LOG_FILE}" \
        || { log "ERROR" "Не удалось создать ${REMOTE_BASE_DIR}"; exit 1; }
fi
log "INFO" "Базовый каталог готов."

# ============ КОПИРОВАНИЕ ФАЙЛОВ ============
TOTAL=0; SUCCESS=0; FAILED=0
FAILED_DIRS=()
TOTAL_NEW=0; TOTAL_UPDATED=0; TOTAL_DELETED=0

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
    log "INFO" "Копирование: ${SRC}  ->  ${DEST}"

    # Формируем опции для отслеживания изменений
    CHANGE_OPTS=""
    if [[ "${USE_CHECKSUM}" == "yes" ]]; then
        CHANGE_OPTS="${CHANGE_OPTS} --checksum"
        log "INFO" "Используется проверка по контрольной сумме (checksum)"
    fi
    
    if [[ "${ITEMIZE_CHANGES}" == "yes" ]]; then
        CHANGE_OPTS="${CHANGE_OPTS} --itemize-changes"
        log "INFO" "Включён детальный список изменений (itemize-changes)"
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

        # Запуск rsync
        ${LOCAL_RSYNC_CMD} -avz --delete \
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
                
                # Генерация отчёта об изменениях
                generate_change_report "${RSYNC_CHANGE_LOG}" "${CHANGE_REPORT}"
                
                # Сохранение истории
                if [[ "${SAVE_CHANGE_HISTORY}" == "yes" ]]; then
                    save_change_history "${RSYNC_CHANGE_LOG}"
                fi
                
                # Получение статистики передачи
                get_transfer_stats "${RSYNC_CHANGE_LOG}" "${STATS_FILE}"
                
                # Подсчёт изменений для общей статистики
                if [[ -f "${RSYNC_CHANGE_LOG}" ]]; then
                    new_files=$(grep -c "^>f\.\.\.\.\.\.\." "${RSYNC_CHANGE_LOG}" 2>/dev/null || echo "0")
                    updated_files=$(grep "^>f" "${RSYNC_CHANGE_LOG}" 2>/dev/null | grep -v "^>f\.\.\.\.\.\.\." | wc -l || echo "0")
                    deleted_files=$(grep -c "^\*deleting" "${RSYNC_CHANGE_LOG}" 2>/dev/null || echo "0")
                    
                    TOTAL_NEW=$((TOTAL_NEW + new_files))
                    TOTAL_UPDATED=$((TOTAL_UPDATED + updated_files))
                    TOTAL_DELETED=$((TOTAL_DELETED + deleted_files))
                    
                    log "INFO" "Изменений: новых=${new_files}, обновлено=${updated_files}, удалено=${deleted_files}"
                fi
                ;;
            3)
                log "ERROR" "Код 3: Ошибка прав доступа (Permission denied)"
                log "ERROR" "Проверьте права на ${REMOTE_BASE_DIR} на удалённом хосте"
                FAILED=$((FAILED + 1))
                FAILED_DIRS+=("${SRC} (Permission denied, code 3)")
                RC=0
                break
                ;;
            127)
                log "ERROR" "Код 127: команда не найдена."
                log "ERROR" "Проверьте пути к rsync локально и на удалённом хосте"
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

# ============ ИТОГОВЫЙ ОТЧЁТ ============
log "INFO" "======================================================"
log "INFO" "ИТОГОВЫЙ ОТЧЁТ"
log "INFO" "======================================================"
log "INFO" "Всего директорий: ${TOTAL}"
log "INFO" "Успешно: ${SUCCESS}"
log "INFO" "С ошибками: ${FAILED}"
log "INFO" ""
log "INFO" "ОБЩАЯ СТАТИСТИКА ИЗМЕНЕНИЙ:"
log "INFO" "  Новых файлов:      ${TOTAL_NEW}"
log "INFO" "  Обновлённых файлов: ${TOTAL_UPDATED}"
log "INFO" "  Удалённых файлов:  ${TOTAL_DELETED}"
log "INFO" "  ВСЕГО изменений:   $((TOTAL_NEW + TOTAL_UPDATED + TOTAL_DELETED))"

if [[ ${#FAILED_DIRS[@]} -gt 0 ]]; then
    log "ERROR" ""
    log "ERROR" "ДИРЕКТОРИИ С ОШИБКАМИ:"
    for D in "${FAILED_DIRS[@]}"; do
        log "ERROR" "  - ${D}"
    done
fi

log "INFO" ""
log "INFO" "ЛОГИ И ОТЧЁТЫ:"
log "INFO" "  Основной лог:      ${LOG_FILE}"
log "INFO" "  Лог изменений:     ${RSYNC_CHANGE_LOG}"
log "INFO" "  Отчёт:             ${CHANGE_REPORT}"
log "INFO" "  Статистика:        ${STATS_FILE}"
log "INFO" "  История:           ${HISTORY_DIR}/"
log "INFO" "======================================================"

[[ ${FAILED} -eq 0 ]] && exit 0 || exit 2
```

## Что добавлено для отслеживания изменений

### 1. Детальный отчёт об изменениях
Скрипт генерирует файл `report_YYYYMMDD_HHMMSS.txt` с:
- Статистикой (новые/обновлённые/удалённые файлы)
- Списком всех изменённых файлов с типом изменения
- Кодовыми обозначениями изменений

### 2. История изменений
Все изменения сохраняются в `/var/log/rsync_transfer/history/history_YYYYMMDD.log` для отслеживания динамики между запусками.

### 3. Статистика передачи
Файл `stats_YYYYMMDD_HHMMSS.txt` содержит подробную статистику rsync (количество файлов, размер, скорость).

### 4. Подсчёт изменений в реальном времени
В основном логе выводится количество новых/обновлённых/удалённых файлов для каждой директории.

### 5. Итоговая статистика
В конце скрипт выводит общую статистику по всем директориям.

## Пример вывода отчёта

```
==================================================
ОТЧЁТ ОБ ИЗМЕНЕНИЯХ
Дата: 2026-10-09 19:15:30
Источник: /lc_file_storage/
Назначение: user001@srv001:/lc_file_storage
==================================================

СТАТИСТИКА:
  Новых файлов:      15
  Обновлённых файлов: 3
  Удалённых файлов:  2
  Новых директорий:  1
  ВСЕГО изменений:   21

==================================================
ДЕТАЛЬНЫЙ СПИСОК ИЗМЕНЕНИЙ:
==================================================

НОВЫЕ ФАЙЛЫ:
  + /lc_file_storage/new_file1.txt
  + /lc_file_storage/new_file2.txt
  ...

ОБНОВЛЁННЫЕ ФАЙЛЫ:
  ~ /lc_file_storage/updated_file.txt
  ...

УДАЛЁННЫЕ ФАЙЛЫ:
  - /lc_file_storage/deleted_file.txt
  ...

==================================================
```

## Коды изменений в логе rsync

- `>f........` - новый файл отправлен
- `>f.st....` - файл обновлён (размер/время)
- `*deleting` - файл удалён на приёмнике
- `>d........` - новая директория создана
- Первая буква: `f` = файл, `d` = директория
- Вторая буква: `.` = без изменений, буква = тип изменения

______
______
______
______
______


Добавление root-прав в скрипт rsync

rsync должен запускаться от root, чтобы:

· читать файлы, недоступные обычному пользователю (например, /var/lib/mysql/, /etc/, чужие ~/.ssh/);
· сохранять владельцев/группы (--numeric-ids работает только под root);
· сохранять права, включая SUID/SGID-биты;
· сохранять device-файлы, симлинки с владельцами;
· создавать каталоги в защищённых местах.

Если запустить под обычным пользователем — rsync тихо «съест» часть файлов (выдаст Permission denied в лог) и на приёмнике будут неверные владельцы. Ниже — исправленный скрипт с проверкой root и правильной работой на обеих сторонах.

---

Что важно понимать про «root на двух сторонах»

Что На источнике (откуда) На приёмнике (куда)
Читать все файлы нужен root —
Сохранять владельцев (-a) нужен root нужен root
Создавать файлы в защищённых каталогах — нужен root
Создавать device/спецфайлы (-D) нужен root нужен root

Значит root нужен на обеих машинах. Это делается одним из двух способов:

1. root@ по SSH на приёмник (обычно запрещён политикой — PermitRootLogin no);
2. обычный пользователь на приёмнике, но с sudo rsync — и --rsync-path="sudo rsync".

В скрипте реализую оба варианта с переключателем.

---

Исправленный скрипт с root-правами

```bash
#!/bin/bash
#
# rsync_transfer_root.sh — перенос данных с root-правами
# Запускать ТОЛЬКО от root (или через sudo).
#

# ============ ГАРАНТИРОВАННЫЙ PATH ============
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:${PATH}"

# ============ НАСТРОЙКИ ============
SOURCE_DIRS=(
    "/data/app1/"
    "/data/app2/"
    "/var/lib/mysql/"
)

REMOTE_USER="backupuser"
REMOTE_HOST="192.168.1.100"
REMOTE_PORT="22"
REMOTE_BASE_DIR="/backup/data"
PASS_FILE="/root/.rsync_pass"

# --- Способ получения root на приёмнике ---
# "user"  — подключаемся как REMOTE_USER, но rsync запускаем через sudo
# "root"  — подключаемся как root@ (требует PermitRootLogin yes)
REMOTE_PRIV_MODE="user"

# Если REMOTE_PRIV_MODE="root" — под каким пользователем идём по SSH:
REMOTE_ROOT_USER="root"

# Путь к sudo на приёмнике (обычно /usr/bin/sudo)
REMOTE_SUDO="/usr/bin/sudo"

# ============ ЛОГИ ============
LOG_DIR="/var/log/rsync_transfer"
LOG_FILE="${LOG_DIR}/rsync_$(date +%Y%m%d_%H%M%S).log"

MAX_RETRIES=3
SSH_TIMEOUT=30

# ============ ПРОВЕРКА ROOT ============
if [[ ${EUID} -ne 0 ]]; then
    echo "ОШИБКА: скрипт должен запускаться от root (или через sudo)."
    echo "Текущий UID: ${EUID}"
    echo "Запустите: sudo $0"
    exit 1
fi

mkdir -p "${LOG_DIR}"
umask 077

log() {
    local level="$1"; shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${level}] $*" | tee -a "${LOG_FILE}"
}

# ============ ПРОВЕРКА БИНАРНИКОВ ============
SSHPASS_BIN="$(command -v sshpass 2>/dev/null)"
SSH_BIN="$(command -v ssh 2>/dev/null)"
RSYNC_BIN="$(command -v rsync 2>/dev/null)"

[[ -z "${SSH_BIN}"   ]] && { echo "ssh не найден";     exit 1; }
[[ -z "${RSYNC_BIN}" ]] && { echo "rsync не найден";   exit 1; }
[[ -z "${SSHPASS_BIN}" ]] && { echo "sshpass не найден"; exit 1; }

log "INFO" "Запуск от: $(id -un) (UID=${EUID})"
log "INFO" "Бинарники: ssh=${SSH_BIN} rsync=${RSYNC_BIN} sshpass=${SSHPASS_BIN}"

# ============ ФОРМИРУЕМ SSH-КОМАНДУ ============
[[ ! -f "${PASS_FILE}" ]] && { log "ERROR" "Нет файла пароля ${PASS_FILE}"; exit 1; }

PERMS=$(stat -c '%a' "${PASS_FILE}" 2>/dev/null || echo "???")
[[ "${PERMS}" != "600" && "${PERMS}" != "400" ]] && \
    log "WARN" "Права на ${PASS_FILE}=${PERMS}, рекомендуется 600."

SSH_OPTS="-p ${REMOTE_PORT} -o ConnectTimeout=${SSH_TIMEOUT} -o StrictHostKeyChecking=accept-new"

# Куда подключаемся по SSH
if [[ "${REMOTE_PRIV_MODE}" == "root" ]]; then
    LOGIN_USER="${REMOTE_ROOT_USER}"
    RSYNC_PATH_OPT=""                       # root и так root
    log "INFO" "Режим: root@ по SSH"
else
    LOGIN_USER="${REMOTE_USER}"
    # На приёмнике rsync запускается через sudo
    RSYNC_PATH_OPT="--rsync-path=${REMOTE_SUDO} ${RSYNC_BIN}"
    log "INFO" "Режим: ${REMOTE_USER}@ + sudo rsync на приёмнике"
fi

DEST_LOGIN="${LOGIN_USER}@${REMOTE_HOST}"

# sshpass-обёртка вокруг ssh (абсолютные пути!)
SSHPASS_SSH="${SSHPASS_BIN} -f ${PASS_FILE} ${SSH_BIN} ${SSH_OPTS}"
# sshpass-обёртка для прямых ssh-команд (проверка соединения, mkdir)
run_remote() { ${SSHPASS_BIN} -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} "${DEST_LOGIN}" "$@"; }

# ============ СТАРТ ============
log "INFO" "======================================================"
log "INFO" "Перенос с root-правами"
log "INFO" "Источник: ${SOURCE_DIRS[*]}"
log "INFO" "Назначение: ${DEST_LOGIN}:${REMOTE_BASE_DIR}"
log "INFO" "======================================================"

# --- Проверка соединения ---
log "INFO" "Проверка соединения..."
if ! run_remote "echo ok" &>>"${LOG_FILE}"; then
    log "ERROR" "Не удалось подключиться к ${REMOTE_HOST}"
    exit 1
fi
log "INFO" "Соединение установлено."

# --- Проверка sudo на приёмнике (если режим user) ---
if [[ "${REMOTE_PRIV_MODE}" == "user" ]]; then
    log "INFO" "Проверка sudo на приёмнике..."
    if ! run_remote "${REMOTE_SUDO} -n id -u" &>>"${LOG_FILE}"; then
        log "ERROR" "sudo на приёмнике требует пароль или запрещён."
        log "ERROR" "Настройте NOPASSWD для rsync или используйте REMOTE_PRIV_MODE=root"
        exit 1
    fi
    SUDO_UID=$(run_remote "${REMOTE_SUDO} -n id -u" 2>/dev/null)
    if [[ "${SUDO_UID}" != "0" ]]; then
        log "ERROR" "sudo на приёмнике не даёт root (UID=${SUDO_UID})"
        exit 1
    fi
    log "INFO" "sudo на приёмнике работает, UID=0"
fi

# --- Создаём базовый каталог (тоже от root!) ---
log "INFO" "Создание ${REMOTE_BASE_DIR} на приёмнике..."
if [[ "${REMOTE_PRIV_MODE}" == "user" ]]; then
    run_remote "${REMOTE_SUDO} -n mkdir -p '${REMOTE_BASE_DIR}'" &>>"${LOG_FILE}" \
        || { log "ERROR" "Не удалось создать ${REMOTE_BASE_DIR}"; exit 1; }
else
    run_remote "mkdir -p '${REMOTE_BASE_DIR}'" &>>"${LOG_FILE}" \
        || { log "ERROR" "Не удалось создать ${REMOTE_BASE_DIR}"; exit 1; }
fi

# ============ ПЕРЕНОС ============
TOTAL=0; SUCCESS=0; FAILED=0
FAILED_DIRS=()

for SRC in "${SOURCE_DIRS[@]}"; do
    TOTAL=$((TOTAL + 1))

    if [[ ! -d "${SRC}" ]]; then
        log "WARN" "Источник не найден: ${SRC}"
        FAILED=$((FAILED + 1))
        FAILED_DIRS+=("${SRC} (не существует)")
        continue
    fi

    # Проверяем читаемость источника под root
    if ! head -c 1 "$(find "${SRC}" -type f -print -quit 2>/dev/null)" &>/dev/null; then
        log "WARN" "Некоторые файлы в ${SRC} могут быть нечитаемы даже root (SELinux/ACL?)"
    fi

    DEST="${DEST_LOGIN}:${REMOTE_BASE_DIR}/$(basename "${SRC}")"
    log "INFO" "------------------------------------------------------"
    log "INFO" "Перенос: ${SRC}  ->  ${DEST}"

    RETRY=0; RC=1
    while [[ ${RETRY} -lt ${MAX_RETRIES} && ${RC} -ne 0 ]]; do
        RETRY=$((RETRY + 1))
        log "INFO" "Попытка ${RETRY}/${MAX_RETRIES}..."

        # Ключевые для root опции:
        #   -a  — рекурсия, права, времена, симлинки, владельцы, группы, устройства
        #   -H  — сохранять жёсткие ссылки
        #   -A  — сохранять ACL
        #   -X  — сохранять расширенные атрибуты (xattr)
        #   --numeric-ids — не маппить UID/GID (обязательно при root-копировании)
        #   -S  — разряженные файлы
        #   -D  — устройства и спецфайлы (эквивалент --devices --specials)
        #   --super         — попытаться применить root-атрибуты на приёмнике
        #   --perms         — явно выставлять права
        "${RSYNC_BIN}" -aHAXS \
            --numeric-ids \
            --devices --specials \
            --super \
            --partial --progress --stats --human-readable \
            --delete \
            ${RSYNC_PATH_OPT:+${RSYNC_PATH_OPT}} \
            -e "${SSHPASS_SSH}" \
            "${SRC}" "${DEST}" &>>"${LOG_FILE}"
        RC=$?

        case ${RC} in
            0)
                log "INFO" "Успешно: ${SRC}"
                SUCCESS=$((SUCCESS + 1))
                ;;
            23)
                # 23 = partial transfer (некоторые файлы пропущены — часто из-за прав)
                log "WARN" "Частичный перенос (код 23). Проверьте права/SELinux в логе."
                SUCCESS=$((SUCCESS + 1))   # считаем успешным, но с замечанием
                ;;
            127)
                log "ERROR" "Код 127: команда не найдена (sshpass/ssh/sudo)."
                FAILED=$((FAILED + 1)); FAILED_DIRS+=("${SRC} (127)"); break
                ;;
            *)
                log "WARN" "Ошибка rsync (код ${RC}) для ${SRC}"
                sleep 5
                ;;
        esac
    done

    if [[ ${RC} -ne 0 && ${RC} -ne 23 ]]; then
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
log "INFO" "Лог: ${LOG_FILE}"

[[ ${FAILED} -eq 0 ]] && exit 0 || exit 2
```

---

Настройка приёмника (одна из двух схем)

Схема A — SSH-логин под обычным пользователем + sudo rsync (рекомендуется)

На приёмнике (192.168.1.100) создаём пользователя backupuser и даём ему право запускать только rsync и служебные команды от root без пароля:

```bash
# На приёмнике от root:
useradd -m -s /bin/bash backupuser
mkdir -p /root/.ssh && chmod 700 /root/.ssh

# Разрешить backupuser запускать rsync/mkdir/id от root без пароля
cat >/etc/sudoers.d/rsync-backup <<'EOF'
backupuser ALL=(root) NOPASSWD: /usr/bin/rsync, /usr/bin/mkdir, /usr/bin/id
EOF
chmod 440 /etc/sudoers.d/rsync-backup
```

Ключевой флаг в скрипте: --rsync-path="sudo rsync" — rsync на приёмной стороне запускается как root, хотя SSH-сессия открыта под backupuser. Именно так работают почти все бэкапы в проде.

Схема B — прямой SSH под root

На приёмнике в /etc/ssh/sshd_config:

```
PermitRootLogin prohibit-password     # только по ключу, пароль запрещён
```

Плюс в скрипте поставьте REMOTE_PRIV_MODE="root". Менее безопасно, но проще.

---

Что именно в скрипте обеспечивает root-доступ

Место Что делает
if [[ ${EUID} -ne 0 ]] в начале Скрипт падает сразу, если запущен не от root
--rsync-path="sudo rsync" rsync на приёмнике стартует под root, сохраняя владельцев
-aHAXS -a + ACL (-A) + xattr (-X) + sparse (-S) + hardlinks (-H)
--numeric-ids UID/GID передаются как есть, не маппятся на пользователя SSH-сессии
--devices --specials Переносятся device-ноды и FIFO/socket-файлы
--super rsync пытается применить root-атрибуты, даже если получатель не root
--delete Удаление на приёмнике — осторожно! требует root там же
Проверка sudo -n id -u Убеждаемся, что sudo реально даёт UID 0 и не просит пароль

---

Проверка, что root-права реально применились

После копирования сравните владельцев на источнике и приёмнике:

```bash
# На источнике:
find /var/lib/mysql -printf '%u:%g %m %p\n' | sort > /tmp/src_own.txt

# На приёмнике:
ssh backupuser@192.168.1.100 "sudo find /backup/data/mysql -printf '%u:%g %m %p\n' | sort" > /tmp/dst_own.txt

diff /tmp/src_own.txt /tmp/dst_own.txt && echo "OK: владельцы и права идентичны"
```

Если всё совпало — значит root-права на обеих сторонах сработали. Если в диффе куча backupuser:backupuser вместо mysql:mysql — значит --rsync-path="sudo rsync" не применился (или sudo требует пароль и тихо упал).

---

Частые ошибки при root-копировании

1. Запустили под sudo, но SSH идёт под непривилегированного пользователя.
Локальный root ≠ удалённый root. Без --rsync-path="sudo rsync" на приёмнике файлы лягут под backupuser.

2. sudo на приёмнике требует пароль.
sudo -n провалится молча. Обязателен NOPASSWD в sudoers.

3. SELinux на приёмнике (RHEL/CentOS/Fedora).
Даже root не сможет создать файл, если контекст запрещает. Либо setenforce 0 на время, либо переносите контексты: добавьте -X (уже есть) и --numeric-ids. В тяжёлых случаях — restorecon -R на приёмнике после копирования.

4. NFS/CIFS на приёмнике.
Владельцы не сохранятся, если ФС не поддерживает chown или смонтирована с no_root_squash. Проверьте mount | grep /backup/data.

5. --delete под root — это опасно вдвойне.
Root удалит файлы, которые обычный пользователь не смог бы. Всегда сначала прогоняйте с --dry-run --delete, смотрите лог, и только потом без -n.

---

Минимальный безопасный запуск

```bash
# 1. Файл с паролем
echo 'ssh-password-of-backupuser' > /root/.rsync_pass
chmod 600 /root/.rsync_pass

# 2. На приёмнике — sudoers NOPASSWD (см. схему A)

# 3. Пробный прогон с dry-run (добавьте -n в rsync вручную)
sudo ./rsync_transfer_root.sh

# 4. Проверка лога
tail -f /var/log/rsync_transfer/rsync_*.log
```

Если по итогу владельцы на приёмнике совпадают с источником, а в логе нет code 23 и Permission denied — root-права настроены правильно.
_____
_____
_____
_____
_____

Локальный root + удалённый root в одном скрипте

Разделим ответственность явно:

Сторона Кто нужен Зачем
Локально (источник) root читать файлы из закрытых каталогов (/var/lib/mysql/, ~/.ssh/, 600-файлы)
Удалённо (приёмник) root сохранять владельцев/группы, писать в защищённые каталоги

Локальный root даёт только чтение источника. Но одной этой половины мало: без root на приёмнике rsync не сможет применить chown/chmod, и файлы лягут под пользователем SSH-сессии. Поэтому в скрипте ниже — оба уровня сразу, с явным авто-повышением прав.

---

Ключевая идея: самоперезапуск через sudo

Чтобы не забыть sudo ./script.sh, добавляем в начало паттерн re-exec:

```bash
if [[ ${EUID} -ne 0 ]]; then
    exec sudo -- "$0" "$@"
fi
```

Скрипт сам поднимется до root, сохранив аргументы. Это и есть «локальный root».

---

Полный скрипт: локальный root + удалённый root

```bash
#!/bin/bash
#
# rsync_transfer_root_both.sh
# Локальный root (чтение источника) + удалённый root (запись приёмника)
#

# ============ ГАРАНТИРОВАННЫЙ PATH ============
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:${PATH}"

# ============================================================
# ЧАСТЬ 1. ЛОКАЛЬНЫЙ ROOT — самоперезапуск через sudo
# ============================================================
if [[ ${EUID} -ne 0 ]]; then
    if command -v sudo &>/dev/null; then
        echo "[*] Повышение прав: перезапуск через sudo..."
        exec sudo -- "$0" "$@"
    else
        echo "ОШИБКА: скрипт требует root, а sudo не установлен."
        echo "Запустите от root: su -c '$0'"
        exit 1
    fi
fi

# Теперь мы точно root локально
log_local_root() {
    echo "[$(date '+%F %T')] [INFO] Локальный пользователь: $(id -un) (UID=${EUID})"
}

# ============ НАСТРОЙКИ ============
SOURCE_DIRS=(
    "/data/app1/"
    "/data/app2/"
    "/var/lib/mysql/"
)

REMOTE_USER="backupuser"
REMOTE_HOST="192.168.1.100"
REMOTE_PORT="22"
REMOTE_BASE_DIR="/backup/data"
PASS_FILE="/root/.rsync_pass"

# Как получаем root на приёмнике:
#   "user"  — SSH под REMOTE_USER + sudo rsync (рекомендуется)
#   "root"  — SSH прямо под root@ (нужен PermitRootLogin)
REMOTE_PRIV_MODE="user"
REMOTE_ROOT_USER="root"
REMOTE_SUDO="/usr/bin/sudo"

LOG_DIR="/var/log/rsync_transfer"
LOG_FILE="${LOG_DIR}/rsync_$(date +%Y%m%d_%H%M%S).log"
MAX_RETRIES=3
SSH_TIMEOUT=30

mkdir -p "${LOG_DIR}"
umask 077

log() {
    local level="$1"; shift
    echo "[$(date '+%F %T')] [${level}] $*" | tee -a "${LOG_FILE}"
}

# ============ ПРОВЕРКА ЛОКАЛЬНОГО ROOT (ещё раз, для наглядности) ============
if [[ ${EUID} -ne 0 ]]; then
    log "ERROR" "Локальный root не получен. Выход."
    exit 1
fi
log "INFO" "Локальный root: OK ($(id -un), UID=${EUID})"

# ============ БИНАРНИКИ ============
SSHPASS_BIN="$(command -v sshpass)"
SSH_BIN="$(command -v ssh)"
RSYNC_BIN="$(command -v rsync)"
[[ -z "${SSHPASS_BIN}" ]] && { log "ERROR" "sshpass не найден"; exit 1; }
[[ -z "${SSH_BIN}"     ]] && { log "ERROR" "ssh не найден";     exit 1; }
[[ -z "${RSYNC_BIN}"   ]] && { log "ERROR" "rsync не найден";   exit 1; }
log "INFO" "Бинарники: ssh=${SSH_BIN}, rsync=${RSYNC_BIN}, sshpass=${SSHPASS_BIN}"

# ============ ПРОВЕРКА ЛОКАЛЬНОГО ДОСТУПА К ИСТОЧНИКАМ ============
# Под root читаем первый байт первого файла каждого каталога —
# если не читается, значит проблема с SELinux/ACL/immutable, а не с правами.
log "INFO" "Проверка локального доступа к источникам..."
for SRC in "${SOURCE_DIRS[@]}"; do
    if [[ ! -d "${SRC}" ]]; then
        log "WARN" "Источник отсутствует: ${SRC}"
        continue
    fi
    FIRST_FILE=$(find "${SRC}" -type f -print -quit 2>/dev/null)
    if [[ -n "${FIRST_FILE}" ]]; then
        if ! dd if="${FIRST_FILE}" of=/dev/null bs=1 count=1 status=none 2>/dev/null; then
            log "WARN" "Не читается даже root: ${FIRST_FILE} (SELinux/ACL/immutable?)"
        else
            log "INFO" "Чтение OK: ${FIRST_FILE}"
        fi
    fi
done

# ============ SSH-ОБЁРТКА ============
[[ ! -f "${PASS_FILE}" ]] && { log "ERROR" "Нет файла пароля ${PASS_FILE}"; exit 1; }
PERMS=$(stat -c '%a' "${PASS_FILE}")
[[ "${PERMS}" != "600" && "${PERMS}" != "400" ]] && \
    log "WARN" "Права ${PASS_FILE}=${PERMS}, рекомендуется 600."

SSH_OPTS="-p ${REMOTE_PORT} -o ConnectTimeout=${SSH_TIMEOUT} -o StrictHostKeyChecking=accept-new"

if [[ "${REMOTE_PRIV_MODE}" == "root" ]]; then
    LOGIN_USER="${REMOTE_ROOT_USER}"
    RSYNC_PATH_OPT=""
    log "INFO" "Приёмник: root@ по SSH"
else
    LOGIN_USER="${REMOTE_USER}"
    # Удалённый rsync запускается через sudo → сохранит владельцев
    RSYNC_PATH_OPT="--rsync-path=${REMOTE_SUDO} ${RSYNC_BIN}"
    log "INFO" "Приёмник: ${REMOTE_USER}@ + sudo rsync"
fi
DEST_LOGIN="${LOGIN_USER}@${REMOTE_HOST}"

SSHPASS_SSH="${SSHPASS_BIN} -f ${PASS_FILE} ${SSH_BIN} ${SSH_OPTS}"
run_remote() {
    ${SSHPASS_BIN} -f "${PASS_FILE}" "${SSH_BIN}" ${SSH_OPTS} "${DEST_LOGIN}" "$@"
}

# ============ ПРОВЕРКА УДАЛЁННОГО ROOT ============
log "INFO" "Проверка соединения с ${REMOTE_HOST}..."
run_remote "echo ok" &>>"${LOG_FILE}" || { log "ERROR" "Нет связи"; exit 1; }

if [[ "${REMOTE_PRIV_MODE}" == "user" ]]; then
    log "INFO" "Проверка sudo на приёмнике..."
    SUDO_UID=$(run_remote "${REMOTE_SUDO} -n id -u" 2>/dev/null)
    if [[ "${SUDO_UID}" != "0" ]]; then
        log "ERROR" "sudo на приёмнике не даёт root (UID='${SUDO_UID}')"
        log "ERROR" "Добавьте NOPASSWD в /etc/sudoers.d/rsync-backup на приёмнике:"
        log "ERROR" "  ${REMOTE_USER} ALL=(root) NOPASSWD: ${REMOTE_SUDO}, ${RSYNC_BIN}, /usr/bin/mkdir"
        exit 1
    fi
    log "INFO" "Удалённый root через sudo: OK (UID=0)"
else
    RUID=$(run_remote "id -u" 2>/dev/null)
    [[ "${RUID}" != "0" ]] && { log "ERROR" "SSH под root@, но UID=${RUID}"; exit 1; }
    log "INFO" "Удалённый root напрямую: OK"
fi

# Создаём базовый каталог от root на приёмнике
log "INFO" "Создание ${REMOTE_BASE_DIR} на приёмнике..."
if [[ "${REMOTE_PRIV_MODE}" == "user" ]]; then
    run_remote "${REMOTE_SUDO} -n mkdir -p '${REMOTE_BASE_DIR}'" &>>"${LOG_FILE}" \
        || { log "ERROR" "Не удалось создать ${REMOTE_BASE_DIR}"; exit 1; }
else
    run_remote "mkdir -p '${REMOTE_BASE_DIR}'" &>>"${LOG_FILE}" \
        || { log "ERROR" "Не удалось создать ${REMOTE_BASE_DIR}"; exit 1; }
fi

# ============ ПЕРЕНОС ============
log "INFO" "======================================================"
log "INFO" "Локальный root → чтение источника"
log "INFO" "Удалённый root  → запись приёмника (владельцы/права)"
log "INFO" "======================================================"

TOTAL=0; SUCCESS=0; FAILED=0
FAILED_DIRS=()

for SRC in "${SOURCE_DIRS[@]}"; do
    TOTAL=$((TOTAL + 1))
    if [[ ! -d "${SRC}" ]]; then
        log "WARN" "Пропуск (нет источника): ${SRC}"
        FAILED=$((FAILED + 1)); FAILED_DIRS+=("${SRC} (нет)")
        continue
    fi

    DEST="${DEST_LOGIN}:${REMOTE_BASE_DIR}/$(basename "${SRC}")"
    log "INFO" "--- ${SRC}  ->  ${DEST}"

    RETRY=0; RC=1
    while [[ ${RETRY} -lt ${MAX_RETRIES} && ${RC} -ne 0 ]]; do
        RETRY=$((RETRY + 1))
        log "INFO" "Попытка ${RETRY}/${MAX_RETRIES}..."

        # Здесь локальный root даёт чтение источника,
        # а ${RSYNC_PATH_OPT} поднимает rsync на приёмнике до root,
        # чтобы -aHAXS с --numeric-ids реально применились.
        "${RSYNC_BIN}" -aHAXS \
            --numeric-ids \
            --devices --specials \
            --super \
            --partial --progress --stats --human-readable \
            ${RSYNC_PATH_OPT} \
            -e "${SSHPASS_SSH}" \
            "${SRC}" "${DEST}" &>>"${LOG_FILE}"
        RC=$?

        case ${RC} in
            0)
                log "INFO" "Успешно: ${SRC}"; SUCCESS=$((SUCCESS + 1)) ;;
            23)
                log "WARN" "Частичный перенос (23) — проверьте лог (права/SELinux)"
                SUCCESS=$((SUCCESS + 1)) ;;
            127)
                log "ERROR" "Код 127 — команда не найдена (sshpass/ssh/sudo)"
                FAILED=$((FAILED + 1)); FAILED_DIRS+=("${SRC} (127)"); break ;;
            *)
                log "WARN" "Ошибка rsync (${RC}) для ${SRC}"; sleep 5 ;;
        esac
    done

    [[ ${RC} -ne 0 && ${RC} -ne 23 ]] && { FAILED=$((FAILED + 1)); FAILED_DIRS+=("${SRC}"); }
done

# ============ ИТОГИ ============
log "INFO" "======================================================"
log "INFO" "ИТОГИ: всего=${TOTAL}, успешно=${SUCCESS}, ошибок=${FAILED}"
for D in "${FAILED_DIRS[@]}"; do log "ERROR" "  - ${D}"; done
log "INFO" "Лог: ${LOG_FILE}"
log "INFO" "Проверка владельцев на приёмнике:"
log "INFO" "  ssh ${LOGIN_USER}@${REMOTE_HOST} 'sudo find ${REMOTE_BASE_DIR} -printf \"%u:%g %m %p\\n\" | head'"

[[ ${FAILED} -eq 0 ]] && exit 0 || exit 2
```

---

Что именно даёт локальный root

1. Авто-повышение прав

```bash
if [[ ${EUID} -ne 0 ]]; then
    exec sudo -- "$0" "$@"
fi
```

Запускаете как обычно — скрипт сам поднимется до root и сохранит все аргументы. Если sudo нет, подскажет su -c.

2. Проверка чтения источника под root

```bash
dd if="${FIRST_FILE}" of=/dev/null bs=1 count=1
```

Это отсекает случай «файл виден, но не читается» — например, SELinux-контекст или chattr +i. Без этой проверки rsync потом выдаст permission denied уже в процессе.

3. Флаги rsync, работающие только под root

Флаг Что делает
-a Рекурсия + права + времена + владельцы + группы + симлинки
-H Жёсткие ссылки
-A POSIX ACL
-X xattr (расширенные атрибуты)
-S Разреженные (sparse) файлы
--devices --specials Устройства и FIFO/socket
--numeric-ids UID/GID не маппятся через пользователя SSH
--super Даже если получатель не root — попытаться применить root-атрибуты

Без локального root rsync тихо «просеет» все файлы с 0600 и 0700 чужих пользователей, а без удалённого root — не сможет их корректно разместить с нужными владельцами.

---

Настройка приёмника (обязательный шаг)

На 192.168.1.100 под root:

```bash
# Пользователь для SSH
useradd -m -s /bin/bash backupuser

# Разрешаем ему ТОЛЬКО rsync/mkdir/id от root без пароля
cat >/etc/sudoers.d/rsync-backup <<'EOF'
backupuser ALL=(root) NOPASSWD: /usr/bin/rsync, /usr/bin/mkdir, /usr/bin/id, /usr/bin/find
EOF
chmod 440 /etc/sudoers.d/rsync-backup
visudo -c    # проверка синтаксиса
```

Ключевая строка в скрипте:

```bash
RSYNC_PATH_OPT="--rsync-path=/usr/bin/sudo /usr/bin/rsync"
```

Именно она превращает rsync на приёмнике в root, даже когда SSH-сессия открыта под backupuser.

---

Проверка, что оба root сработали

```bash
# На источнике — от root:
find /var/lib/mysql -printf '%u:%g %m %p\n' | sort > /tmp/src_own.txt

# На приёмнике — через sudo:
ssh backupuser@192.168.1.100 \
    "sudo find /backup/data/mysql -printf '%u:%g %m %p\n' | sort" > /tmp/dst_own.txt

diff /tmp/src_own.txt /tmp/dst_own.txt && echo "OK: владельцы/права идентичны"
```

Ожидаемые признаки проблем:

Симптом Причина
Владельцы на приёмнике = backupuser:backupuser вместо mysql:mysql --rsync-path=sudo rsync не применился
В логе permission denied (13) на источник Локальный root не получен (или SELinux/ACL)
sudo: a password is required Нет NOPASSWD в sudoers на приёмнике
Много code 23 Проблема с правами с одной из сторон
code 127 sshpass/ssh/sudo не найдены в PATH

---

Как это запускается на практике

```bash
# 1. Один раз: файл с SSH-паролем
echo 'ssh-pass-backupuser' > /root/.rsync_pass
chmod 600 /root/.rsync_pass

# 2. Один раз на приёмнике: sudoers NOPASSWD

# 3. Запуск — просто от своего пользователя, скрипт сам поднимется до root
./rsync_transfer_root_both.sh

# Или явно:
sudo ./rsync_transfer_root_both.sh
```

Скрипт сам:

1. Проверит EUID, при необходимости перезапустится через sudo;
2. Убедится, что локально читает источники под root;
3. Проверит, что на приёмнике sudo rsync реально даёт UID=0;
4. Запустит rsync с полным набором root-флагов (-aHAXS --numeric-ids --super).

В итоге получаете точную копию источника — с владельцами, правами, ACL, xattr и hardlink-ами, а не «файлы под backupuser с урезанными правами», как при обычном запуске без root.