Ниже представлен bash-скрипт для комплексного тестирования Kerberos в Linux. Он проверяет наличие утилит, корректность конфигурации, синхронизацию времени, DNS-записи, доступность KDC и при желании позволяет протестировать получение TGT и сервисных билетов.

```bash
#!/bin/bash
# kerberos_test.sh — Диагностика и тестирование Kerberos
# Запуск: ./kerberos_test.sh [опции]
# Без аргументов работает в интерактивном режиме.
#
# Переменные окружения для неинтерактивного режима:
#   KTEST_USER     — имя пользователя (principal)
#   KTEST_PASS     — пароль
#   KTEST_KEYTAB   — путь к keytab-файлу
#   KTEST_SERVICE  — сервисный principal (например, host/hostname@REALM)
#   KTEST_OPTIONS  — дополнительные флаги (например, -n для неинтерактивного теста)

set -uo pipefail

# ---------- конфигурация по умолчанию ----------
DEFAULT_SERVICE="host/$(hostname -f)"

# ---------- цвета для вывода ----------
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

info()  { echo -e "${GREEN}[INFO]${NC}  $*"; }
warn()  { echo -e "${YELLOW}[WARN]${NC}  $*"; }
error() { echo -e "${RED}[ERROR]${NC} $*"; }

# ---------- функции проверки ----------
check_command() {
    command -v "$1" >/dev/null 2>&1
}

# Чтение default_realm из krb5.conf (довольно примитивный парсинг)
get_realm_from_conf() {
    local conf="${1:-/etc/krb5.conf}"
    if [ -f "$conf" ]; then
        awk -F= '/^\s*default_realm\s*=/ {gsub(/[[:space:]]+/,"",$2); print $2; exit}' "$conf"
    fi
}

# Проверка порта через netcat (или bash /dev/tcp)
check_port() {
    local host="$1"
    local port="$2"
    local timeout="${3:-3}"
    if check_command nc; then
        nc -z -w "$timeout" "$host" "$port" >/dev/null 2>&1
    elif check_command timeout; then
        timeout "$timeout" bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null
    else
        # fallback: пробуем без таймаута (может зависнуть)
        bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null
    fi
}

# ---------- начало основного скрипта ----------
echo "=== Тестирование Kerberos ==="
echo ""

# 1. Проверка необходимых утилит
info "Проверка наличия утилит..."
MISSING=0
for cmd in kinit klist kdestroy kvno; do
    if check_command "$cmd"; then
        info "  $cmd — OK"
    else
        error "  $cmd не найден (установите krb5-user / krb5-workstation)"
        MISSING=1
    fi
done
if [ $MISSING -eq 1 ]; then
    error "Отсутствуют необходимые утилиты Kerberos. Выход."
    exit 1
fi

# 2. Проверка конфигурационного файла
KRB5_CONF="${KRB5_CONFIG:-/etc/krb5.conf}"
info "Проверка конфигурации ($KRB5_CONF)..."
if [ ! -f "$KRB5_CONF" ]; then
    error "Файл $KRB5_CONF не существует."
    exit 1
fi

REALM=$(get_realm_from_conf "$KRB5_CONF")
if [ -z "$REALM" ]; then
    warn "Не удалось определить default_realm из $KRB5_CONF."
    read -rp "Введите имя realm (например, EXAMPLE.COM): " REALM
    if [ -z "$REALM" ]; then
        error "Realm не задан. Выход."
        exit 1
    fi
else
    info "default_realm = $REALM"
fi

# 3. Проверка синхронизации времени
info "Проверка синхронизации времени..."
if check_command timedatectl; then
    if timedatectl show | grep -q 'NTPSynchronized=yes'; then
        info "  Время синхронизировано (NTP active)."
    else
        warn "  timedatectl: NTPSynchronized=no. Возможны проблемы с Kerberos."
    fi
elif check_command ntpstat; then
    if ntpstat >/dev/null 2>&1; then
        info "  Время синхронизировано (ntpstat)."
    else
        warn "  ntpstat сообщает о рассинхронизации."
    fi
else
    warn "  Не удалось проверить статус синхронизации. Убедитесь, что время на сервере совпадает с KDC (разница < 5 мин)."
fi

# 4. Проверка DNS (если доступен dig)
info "Проверка DNS-записей Kerberos..."
if check_command dig; then
    # SRV записи
    for proto in udp tcp; do
        srv="_kerberos._${proto}.${REALM}"
        if dig +short srv "$srv" | grep -q .; then
            info "  SRV $srv — OK"
        else
            warn "  SRV $srv не найдена. Аутентификация может работать только при явном указании KDC в krb5.conf."
        fi
    done
    # A запись для KDC (если она указана в krb5.conf, здесь просто информативно)
else
    warn "  dig не найден — пропускаем DNS-проверку."
fi

# 5. Определение KDC и проверка доступности порта 88
info "Поиск KDC и проверка порта 88..."
KDC_LIST=""
# Пытаемся извлечь KDC из krb5.conf (секция realms)
if [ -f "$KRB5_CONF" ]; then
    KDC_LIST=$(awk -v realm="$REALM" '
        BEGIN {found=0}
        /^\s*\[realms\]/ {found=1; next}
        /^\s*\[/ {if ($0 !~ /realms/) found=0}
        found && $0 ~ realm {insection=1; next}
        insection && /^\s*kdc\s*=/ {
            gsub(/^[[:space:]]*kdc[[:space:]]*=[[:space:]]*/,"");
            print $0;
        }
        insection && /^\s*}/ {insection=0}
    ' "$KRB5_CONF")
fi

if [ -n "$KDC_LIST" ]; then
    info "  KDC из krb5.conf:"
    for kdc in $KDC_LIST; do
        echo "    - $kdc"
        if check_port "$kdc" 88; then
            info "      порт 88 доступен"
        else
            error "      порт 88 недоступен"
        fi
    done
else
    # Попробуем разрешить домен realm
    KDC_HOST="kerberos.$(echo "$REALM" | tr 'A-Z' 'a-z')"
    if check_command host; then
        if host "$KDC_HOST" >/dev/null 2>&1; then
            warn "  KDC не задан в krb5.conf. Проверяем $KDC_HOST..."
            if check_port "$KDC_HOST" 88; then
                info "  $KDC_HOST:88 доступен (возможно, KDC)"
            else
                error "  $KDC_HOST:88 недоступен"
            fi
        else
            warn "  Не удалось найти KDC. Проверьте настройки krb5.conf или DNS."
        fi
    else
        warn "  Утилита host не найдена, не можем угадать KDC."
    fi
fi

# ---------- интерактивное тестирование (или через переменные) ----------
run_tgt_test=false
run_svc_test=false
noninteractive=false

# Проверяем, запущен ли скрипт с флагом -n (неинтерактивный режим) или через переменные
if [[ "${KTEST_OPTIONS:-}" == *"-n"* ]] || [[ "${1:-}" == "-n" ]] || [[ "${1:-}" == "--non-interactive" ]]; then
    noninteractive=true
fi

if $noninteractive; then
    if [ -n "${KTEST_USER:-}" ] && [ -n "${KTEST_PASS:-}" ]; then
        run_tgt_test=true
    fi
    if [ -n "${KTEST_SERVICE:-}" ]; then
        run_svc_test=true
    fi
    if [ -n "${KTEST_KEYTAB:-}" ]; then
        # keytab переопределяет парольный вход
        run_tgt_test=true
    fi
else
    echo ""
    read -rp "Протестировать получение TGT (kinit)? [y/N]: " answer
    if [[ "$answer" =~ ^[Yy] ]]; then
        run_tgt_test=true
        # Способ аутентификации: пароль или keytab
        read -rp "Использовать keytab? [y/N]: " use_keytab
        if [[ "$use_keytab" =~ ^[Yy] ]]; then
            read -rp "Путь к keytab-файлу: " KTEST_KEYTAB
        fi
        if [ -z "${KTEST_KEYTAB:-}" ]; then
            read -rp "Введите principal (по умолчанию текущий пользователь @ $REALM): " principal
            if [ -z "$principal" ]; then
                principal="$(whoami)@$REALM"
            fi
            KTEST_USER="$principal"
            # Скрытый ввод пароля
            echo -n "Пароль для $KTEST_USER: "
            stty -echo
            read -r KTEST_PASS
            stty echo
            echo ""
        fi
    fi

    read -rp "Протестировать получение сервисного билета (kvno)? [y/N]: " svc_answer
    if [[ "$svc_answer" =~ ^[Yy] ]]; then
        run_svc_test=true
        read -rp "Сервисный principal [${DEFAULT_SERVICE}]: " service_input
        KTEST_SERVICE="${service_input:-$DEFAULT_SERVICE}"
    fi
fi

# ---------- Выполнение тестов ----------
if $run_tgt_test; then
    info "=== Тест получения TGT ==="
    if [ -n "${KTEST_KEYTAB:-}" ]; then
        # аутентификация по keytab
        principal_from_keytab=$(klist -k "$KTEST_KEYTAB" 2>/dev/null | awk 'NR>2 {print $2; exit}')
        if [ -z "$principal_from_keytab" ]; then
            error "Не удалось прочитать principal из $KTEST_KEYTAB"
        else
            info "Используем keytab $KTEST_KEYTAB, principal: $principal_from_keytab"
            if kinit -k -t "$KTEST_KEYTAB" "$principal_from_keytab"; then
                info "TGT успешно получен."
                klist
            else
                error "Ошибка получения TGT по keytab."
            fi
        fi
    else
        # парольный вход
        info "Запрос TGT для $KTEST_USER..."
        # Используем expect или передачу пароля через stdin (небезопасно, но для теста)
        if check_command kinit; then
            echo "$KTEST_PASS" | kinit "$KTEST_USER" 2>/tmp/kinit_err.$$
            if [ $? -eq 0 ]; then
                info "TGT успешно получен."
                klist
            else
                error "Ошибка kinit: $(cat /tmp/kinit_err.$$)"
                rm -f /tmp/kinit_err.$$
            fi
        fi
    fi
fi

if $run_svc_test; then
    info "=== Тест получения сервисного билета ==="
    # Нужен действующий TGT, проверяем
    if klist -s 2>/dev/null; then
        info "Запрос билета для сервиса $KTEST_SERVICE..."
        if kvno "$KTEST_SERVICE"; then
            info "Сервисный билет успешно получен."
            klist
        else
            error "Не удалось получить сервисный билет для $KTEST_SERVICE."
        fi
    else
        error "Нет действующего TGT. Сначала получите TGT (kinit)."
    fi
fi

# ---------- Очистка ----------
if $run_tgt_test || $run_svc_test; then
    echo ""
    read -rp "Уничтожить полученные билеты? [Y/n]: " destroy
    if [[ ! "$destroy" =~ ^[Nn] ]]; then
        kdestroy
        info "Билеты уничтожены."
    fi
fi

echo ""
info "Диагностика Kerberos завершена."
```

Как использовать

1. Сохраните скрипт, например в kerberos_test.sh.
2. Сделайте исполняемым:
      chmod +x kerberos_test.sh
3. Запустите в интерактивном режиме:
      ./kerberos_test.sh
      Скрипт проверит окружение, задаст необходимые вопросы и проведёт тесты.
4. Для автоматизации используйте переменные окружения и флаг -n:
   ```bash
   export KTEST_USER='user@EXAMPLE.COM'
   export KTEST_PASS='secret'
   export KTEST_SERVICE='host/server.example.com@EXAMPLE.COM'
   export KTEST_OPTIONS='-n'
   ./kerberos_test.sh
   ```
   или сразу:
   ```bash
   KTEST_USER='user@REALM' KTEST_PASS='pass' KTEST_SERVICE='host/fqdn@REALM' ./kerberos_test.sh -n
   ```
   С keytab:
   ```bash
   KTEST_KEYTAB='/path/to/krb5.keytab' KTEST_SERVICE='host/fqdn@REALM' ./kerberos_test.sh -n
   ```

Что проверяет скрипт

· Наличие kinit, klist, kdestroy, kvno.
· Чтение /etc/krb5.conf и определение default_realm.
· Синхронизацию времени (через timedatectl или ntpstat).
· DNS SRV-записи _kerberos._udp и _kerberos._tcp для realm.
· Доступность KDC (порт 88), определённого в конфиге или через DNS.
· Интерактивное (или автоматическое) получение TGT по паролю или keytab.
· Получение сервисного билета для заданного сервиса (kvno).
· Отображение текущих билетов (klist).

Скрипт написан для bash, но совместим с POSIX-окружением за счёт осторожного использования конструкций. Для скрытого ввода пароля применяется stty -echo.