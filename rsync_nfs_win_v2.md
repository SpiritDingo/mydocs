# Настройка синхронизации между SUSE Linux и Windows с NFS и правильными правами доступа

## Полная настройка доступа к файлам при синхронизации

### 1. Настройка Windows NFS сервера с правильными правами

1. **Установка NFS сервера**:
   - В "Управлении сервером" добавьте роль "Служба NFS" → "Сервер NFS"

2. **Настройка общей папки**:
   - Правой кнопкой на папке → Свойства → NFS Sharing
   - Включите "Share this folder"
   - Установите:
     - "Allow anonymous access" (анонимный доступ)
     - "Enable unmapped user access" (доступ без сопоставления пользователей)
     - "Unmapped user access" → "Allow unmapped access" (разрешить неаутентифицированный доступ)
     - "Permissions" → "Read-Write" для всех

3. **Настройка идентификаторов**:
   ```powershell
   Set-NfsMappingStore -EnableLdapLookup $false
   Set-NfsServerConfiguration -EnableAnonymousAccess $true -AnonymousUid 1000 -AnonymousGid 1000
   ```
   (1000 - замените на UID/GID пользователя Linux)

### 2. Настройка SUSE Linux для работы с Windows NFS

1. **Установка компонентов**:
   ```bash
   sudo zypper install nfs-client rpcbind rsync
   ```

2. **Монтирование с правильными правами**:
   ```bash
   sudo mkdir -p /mnt/windows_nfs
   sudo mount -t nfs -o vers=3,nolock,uid=1000,gid=1000 windows_ip:/share_name /mnt/windows_nfs
   ```
   Где:
   - `uid=1000,gid=1000` - замените на ID вашего пользователя Linux (`id -u username`, `id -g username`)
   - `windows_ip` - IP адрес Windows сервера
   - `share_name` - имя общей папки на Windows

3. **Автомонтирование (fstab)**:
   ```bash
   sudo nano /etc/fstab
   ```
   Добавьте строку:
   ```
   windows_ip:/share_name /mnt/windows_nfs nfs vers=3,nolock,uid=1000,gid=1000 0 0
   ```

### 3. Настройка rsync с сохранением прав

1. **Базовая команда синхронизации**:
   ```bash
   rsync -av --chown=USERNAME:GROUPNAME --progress /path/to/source/ /mnt/windows_nfs/destination/
   ```
   Где `USERNAME:GROUPNAME` - пользователь и группа, которые должны владеть файлами

2. **Альтернативный вариант (если проблемы с правами)**:
   ```bash
   rsync -rlptDv --progress --no-o --no-g --chmod=ugo=rwX /path/to/source/ /mnt/windows_nfs/destination/
   ```

### 4. Решение проблем с правами

1. **Проверка текущих прав**:
   ```bash
   ls -la /mnt/windows_nfs/
   ```

2. **Принудительная установка прав**:
   ```bash
   sudo chmod -R 775 /mnt/windows_nfs/
   sudo chown -R username:groupname /mnt/windows_nfs/
   ```

3. **Если файлы создаются с неправильными правами**:
   ```bash
   umask 0002
   rsync -avz --progress --delete /path/to/source/ /mnt/windows_nfs/destination/
   ```

### 5. Дополнительные настройки безопасности

1. **Ограничение доступа по IP**:
   На Windows в PowerShell:
   ```powershell
   Grant-NfsSharePermission -Name "share_name" -ClientName "linux_ip" -Permission "ReadWrite"
   ```

2. **Логирование операций rsync**:
   ```bash
   rsync -avz --log-file=/var/log/rsync_windows.log /path/to/source/ /mnt/windows_nfs/destination/
   ```

### 6. Альтернатива: Samba вместо NFS

Если продолжаются проблемы с правами:
```bash
sudo zypper install cifs-utils
sudo mount -t cifs //windows_ip/share_name /mnt/windows_share -o username=winuser,password=winpass,uid=1000,gid=1000
```

В этом случае используйте аналогичные команды rsync с монтированной Samba-папкой.