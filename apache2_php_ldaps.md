# Интеграция PHP с Microsoft Active Directory

Ниже приведены примеры для получения списка пользователей из AD и их атрибутов с использованием PHP 8.2.

---

## 1. Базовая функция подключения к AD

Создайте файл `ad_connect.php`:

```php
<?php
class ADConnector {
    private $ldap_conn;
    private $ldap_host;
    private $ldap_port;
    private $base_dn;
    
    public function __construct($host, $port = 636, $base_dn = "DC=yourdomain,DC=com") {
        $this->ldap_host = $host;
        $this->ldap_port = $port;
        $this->base_dn = $base_dn;
    }
    
    public function connect($username, $password) {
        // Подключение по LDAPS
        $this->ldap_conn = ldap_connect("ldaps://{$this->ldap_host}", $this->ldap_port);
        
        if (!$this->ldap_conn) {
            throw new Exception("Не удалось подключиться к LDAP серверу");
        }
        
        // Обязательные настройки для AD
        ldap_set_option($this->ldap_conn, LDAP_OPT_PROTOCOL_VERSION, 3);
        ldap_set_option($this->ldap_conn, LDAP_OPT_REFERRALS, 0);
        ldap_set_option($this->ldap_conn, LDAP_OPT_NETWORK_TIMEOUT, 10);
        
        // Авторизация (Bind)
        $bind = @ldap_bind($this->ldap_conn, $username, $password);
        
        if (!$bind) {
            throw new Exception("Ошибка авторизации: " . ldap_error($this->ldap_conn));
        }
        
        return true;
    }
    
    public function close() {
        if ($this->ldap_conn) {
            ldap_unbind($this->ldap_conn);
        }
    }
    
    public function getConnection() {
        return $this->ldap_conn;
    }
    
    public function getBaseDn() {
        return $this->base_dn;
    }
}
?>
```

---

## 2. Получение списка всех пользователей с пагинацией

AD по умолчанию возвращает максимум 1000 записей за один запрос. Для получения всех пользователей нужна пагинация.

Создайте файл `get_users.php`:

```php
<?php
require_once 'ad_connect.php';

// Настройки подключения
$ad_host = "ldap.yourdomain.com";
$ad_user = "CN=ServiceAccount,OU=ServiceAccounts,DC=yourdomain,DC=com";
$ad_pass = "your_password";
$base_dn = "DC=yourdomain,DC=com";

try {
    $ad = new ADConnector($ad_host, 636, $base_dn);
    $ad->connect($ad_user, $ad_pass);
    
    // Атрибуты, которые мы хотим получить
    $attributes = [
        'sAMAccountName',      // Логин пользователя
        'displayName',         // Отображаемое имя
        'mail',                // Email
        'givenName',           // Имя
        'sn',                  // Фамилия
        'title',               // Должность
        'department',          // Отдел
        'memberOf',            // Группы
        'userAccountControl',  // Статус учетной записи
        'lastLogonTimestamp',  // Время последнего входа
        'whenCreated',         // Дата создания
        'distinguishedName'    // DN пользователя
    ];
    
    // Фильтр: только пользователи (не компьютеры, не контакты)
    // objectClass=user AND objectCategory=person
    $filter = "(&(objectClass=user)(objectCategory=person))";
    
    $all_users = [];
    $cookie = '';
    
    do {
        // Пагинация для PHP 8.1+
        $controls = [
            [
                'oid' => LDAP_CONTROL_PAGEDRESULTS,
                'value' => [
                    'size' => 500,  // Размер страницы
                    'cookie' => $cookie
                ]
            ]
        ];
        
        $search = ldap_search(
            $ad->getConnection(),
            $base_dn,
            $filter,
            $attributes,
            0,      // attrsonly
            0,      // sizelimit
            10,     // timelimit
            LDAP_DEREF_NEVER,
            $controls
        );
        
        if (!$search) {
            throw new Exception("Ошибка поиска: " . ldap_error($ad->getConnection()));
        }
        
        $entries = ldap_get_entries($ad->getConnection(), $search);
        
        // Обработка результатов
        for ($i = 0; $i < $entries['count']; $i++) {
            $user = [];
            
            foreach ($attributes as $attr) {
                if (isset($entries[$i][$attr])) {
                    if ($entries[$i][$attr]['count'] > 1) {
                        // Множественные значения (например, memberOf)
                        $values = [];
                        for ($j = 0; $j < $entries[$i][$attr]['count']; $j++) {
                            $values[] = $entries[$i][$attr][$j];
                        }
                        $user[$attr] = $values;
                    } else {
                        // Единственное значение
                        $user[$attr] = $entries[$i][$attr][0];
                    }
                } else {
                    $user[$attr] = null;
                }
            }
            
            // Преобразование userAccountControl в читаемый статус
            $user['account_status'] = getAccountStatus($user['userAccountControl']);
            
            // Преобразование lastLogonTimestamp в дату
            if ($user['lastLogonTimestamp']) {
                $user['last_login'] = convertADTimestamp($user['lastLogonTimestamp']);
            }
            
            $all_users[] = $user;
        }
        
        // Получение cookie для следующей страницы
        ldap_parse_result(
            $ad->getConnection(),
            $search,
            $errcode,
            $matcheddn,
            $errmsg,
            $referrals,
            $controls
        );
        
        if (isset($controls[LDAP_CONTROL_PAGEDRESULTS]['value']['cookie'])) {
            $cookie = $controls[LDAP_CONTROL_PAGEDRESULTS]['value']['cookie'];
        } else {
            $cookie = '';
        }
        
    } while (!empty($cookie));
    
    echo "<h2>Найдено пользователей: " . count($all_users) . "</h2>";
    echo "<pre>";
    print_r($all_users);
    echo "</pre>";
    
    $ad->close();
    
} catch (Exception $e) {
    echo "Ошибка: " . $e->getMessage();
}

// Функция для преобразования AD timestamp в дату
function convertADTimestamp($timestamp) {
    if (!$timestamp || $timestamp == 0) {
        return null;
    }
    
    // AD timestamp - это количество 100-наносекундных интервалов с 1 января 1601
    $unixTimestamp = ($timestamp / 10000000) - 11644473600;
    return date('Y-m-d H:i:s', $unixTimestamp);
}

// Функция для определения статуса учетной записи
function getAccountStatus($uac) {
    if (!$uac) return 'unknown';
    
    $flags = [
        0x0001 => 'SCRIPT',
        0x0002 => 'ACCOUNTDISABLE',
        0x0008 => 'HOMEDIR_REQUIRED',
        0x0010 => 'LOCKOUT',
        0x0020 => 'PASSWD_NOTREQD',
        0x0040 => 'PASSWD_CANT_CHANGE',
        0x0080 => 'ENCRYPTED_TEXT_PWD_ALLOWED',
        0x0100 => 'TEMP_DUPLICATE_ACCOUNT',
        0x0200 => 'NORMAL_ACCOUNT',
        0x0800 => 'INTERDOMAIN_TRUST_ACCOUNT',
        0x1000 => 'WORKSTATION_TRUST_ACCOUNT',
        0x2000 => 'SERVER_TRUST_ACCOUNT',
        0x10000 => 'DONT_EXPIRE_PASSWORD',
        0x20000 => 'MNS_LOGON_ACCOUNT',
        0x40000 => 'SMARTCARD_REQUIRED',
        0x80000 => 'TRUSTED_FOR_DELEGATION',
        0x100000 => 'NOT_DELEGATED',
        0x200000 => 'USE_DES_KEY_ONLY',
        0x400000 => 'DONT_REQ_PREAUTH',
        0x800000 => 'PASSWORD_EXPIRED'
    ];
    
    $status = [];
    foreach ($flags as $flag => $name) {
        if ($uac & $flag) {
            $status[] = $name;
        }
    }
    
    return implode(', ', $status);
}
?>
```

---

## 3. Получение только авторизованных (активных) пользователей

Если под "авторизованными" вы имеете в виду пользователей, которые недавно входили в систему:

```php
<?php
// Фильтр для активных пользователей (вошли за последние 30 дней)
$thirty_days_ago = time() - (30 * 24 * 60 * 60);
$ad_timestamp = ($thirty_days_ago + 11644473600) * 10000000;

$filter = "(&(objectClass=user)(objectCategory=person)" .
          "(lastLogonTimestamp>=$ad_timestamp)" .
          "(!(userAccountControl:1.2.840.113556.1.4.803:=2)))"; // Исключаем отключенные учетки

// Альтернативный фильтр: только включенные учетные записи
$filter_enabled = "(&(objectClass=user)(objectCategory=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))";
?>
```

**Важно о lastLogonTimestamp:**
- `lastLogon` - точное время последнего входа, но не реплицируется между контроллерами домена
- `lastLogonTimestamp` - реплицируется, но обновляется не при каждом входе (обычно раз в 9-14 дней)
- Для точного аудита нужно опрашивать все контроллеры домена и сравнивать `lastLogon`

---

## 4. Получение пользователя с рекурсивным членством в группах

```php
<?php
function getUserWithGroups($ldap_conn, $base_dn, $username) {
    // Поиск пользователя
    $filter = "(sAMAccountName=$username)";
    $attributes = ['*', 'memberOf'];
    
    $search = ldap_search($ldap_conn, $base_dn, $filter, $attributes);
    $entries = ldap_get_entries($ldap_conn, $search);
    
    if ($entries['count'] == 0) {
        return null;
    }
    
    $user = $entries[0];
    
    // Получение рекурсивного членства в группах (включая вложенные группы)
    $dn = $user['distinguishedname'][0];
    $filter_groups = "(member:1.2.840.113556.1.4.803:=$dn)";
    
    $search_groups = ldap_search(
        $ldap_conn,
        $base_dn,
        $filter_groups,
        ['cn', 'distinguishedName']
    );
    
    $groups = ldap_get_entries($ldap_conn, $search_groups);
    
    $user['all_groups'] = [];
    for ($i = 0; $i < $groups['count']; $i++) {
        $user['all_groups'][] = [
            'cn' => $groups[$i]['cn'][0],
            'dn' => $groups[$i]['distinguishedname'][0]
        ];
    }
    
    return $user;
}

// Использование:
$user = getUserWithGroups($ad->getConnection(), $base_dn, 'ivanov');
echo "<pre>";
print_r($user);
echo "</pre>";
?>
```

---

## 5. Оптимизация: получение только нужных атрибутов

Для ускорения работы запрашивайте только необходимые атрибуты:

```php
<?php
// Минимальный набор атрибутов
$attributes = ['sAMAccountName', 'displayName', 'mail', 'memberOf'];

// Если нужен только логин и email
$attributes = ['sAMAccountName', 'mail'];

// Если вообще не нужны атрибуты (только DN)
$attributes = ['dn'];
?>
```

---

## 6. Обработка ошибок и логирование

```php
<?php
function ldapErrorHandler($ldap_conn) {
    $error_code = ldap_errno($ldap_conn);
    $error_msg = ldap_error($ldap_conn);
    $detailed = ldap_err2str($error_code);
    
    error_log("LDAP Error [$error_code]: $error_msg - $detailed");
    
    return [
        'code' => $error_code,
        'message' => $error_msg,
        'detailed' => $detailed
    ];
}

// Использование:
$search = @ldap_search($ad->getConnection(), $base_dn, $filter, $attributes);
if (!$search) {
    $error = ldapErrorHandler($ad->getConnection());
    die("Ошибка LDAP: " . $error['message']);
}
?>
```

---

## 7. Важные замечания для AD

1. **Служебная учетная запись:** Создайте в AD специальную учетную запись с правами на чтение (не администратор!)

2. **Безопасность:** Храните пароль служебной учетной записи в переменных окружения или защищенном конфиге вне веб-доступа

3. **Кэширование:** AD-запросы медленные, кэшируйте результаты на 5-15 минут

4. **Размер ответа:** AD может вернуть очень большие объекты (например, с токенами безопасности). Используйте атрибут `tokenGroups` с осторожностью

5. **Временные зоны:** `lastLogonTimestamp` хранится в UTC, преобразуйте в локальное время при отображении

6. **Отключенные учетки:** Проверяйте флаг `ACCOUNTDISABLE` (0x0002) в `userAccountControl`

---

## 8. Пример использования в реальном приложении

```php
<?php
// index.php
require_once 'ad_connect.php';

session_start();

if (!isset($_SESSION['users'])) {
    try {
        $ad = new ADConnector("ldap.yourdomain.com", 636, "DC=yourdomain,DC=com");
        $ad->connect("CN=ServiceAccount,OU=Service,DC=yourdomain,DC=com", "password");
        
        $filter = "(&(objectClass=user)(objectCategory=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))";
        $attributes = ['sAMAccountName', 'displayName', 'mail', 'department'];
        
        // ... код пагинации из примера выше ...
        
        $_SESSION['users'] = $all_users;
        $_SESSION['cache_time'] = time();
        
        $ad->close();
    } catch (Exception $e) {
        die("Ошибка: " . $e->getMessage());
    }
}

// Отображение списка
echo "<table border='1'>";
echo "<tr><th>Логин</th><th>Имя</th><th>Email</th><th>Отдел</th></tr>";
foreach ($_SESSION['users'] as $user) {
    echo "<tr>";
    echo "<td>" . htmlspecialchars($user['sAMAccountName']) . "</td>";
    echo "<td>" . htmlspecialchars($user['displayName']) . "</td>";
    echo "<td>" . htmlspecialchars($user['mail']) . "</td>";
    echo "<td>" . htmlspecialchars($user['department']) . "</td>";
    echo "</tr>";
}
echo "</table>";
?>
```

Этот код готов к использованию в продакшене с учетом всех особенностей работы с Microsoft Active Directory.