# Установка и настройка синхронизации между SUSE Linux и Windows с использованием rsync и NFS

## 1. Подготовка Windows (NFS сервер)

### 1.1. Установка NFS сервера на Windows
1. Откройте "Управление сервером" → "Добавить роли и компоненты"
2. Выберите "Служба NFS" → "Сервер NFS"
3. Завершите установку

### 1.2. Настройка общего доступа NFS
1. Правой кнопкой на папке → Свойства → Вкладка "NFS Sharing"
2. Нажмите "Manage NFS Sharing"
3. Настройте параметры:
   - Разрешите анонимный доступ (Anonymous UID/GID)
   - Установите флажок "Allow root access"
   - Укажите разрешения (Read/Write)

## 2. Настройка SUSE Linux (NFS клиент)

### 2.1. Установка необходимых пакетов
```bash
sudo zypper install nfs-client rpcbind rsync
```

### 2.2. Монтирование NFS из Windows
```bash
sudo mkdir -p /mnt/windows_nfs
sudo mount -t nfs -o vers=3,nolock windows_ip:/share_name /mnt/windows_nfs
```

Для автоматического монтирования добавьте в `/etc/fstab`:
```
windows_ip:/share_name  /mnt/windows_nfs  nfs  vers=3,nolock  0  0
```

## 3. Настройка синхронизации с помощью rsync

### 3.1. Синхронизация из Linux в Windows
```bash
sudo rsync -avz --progress --delete /path/to/linux/source/ /mnt/windows_nfs/destination/
```

### 3.2. Синхронизация из Windows в Linux
```bash
sudo rsync -avz --progress --delete /mnt/windows_nfs/source/ /path/to/linux/destination/
```

## 4. Автоматизация синхронизации

### 4.1. Создание скрипта синхронизации
```bash
sudo nano /usr/local/bin/sync_windows.sh
```
Содержимое:
```bash
#!/bin/bash
rsync -avz --delete /path/to/linux/source/ /mnt/windows_nfs/destination/ >> /var/log/rsync_windows.log
```

### 4.2. Настройка прав
```bash
sudo chmod +x /usr/local/bin/sync_windows.sh
```

### 4.3. Добавление в cron
```bash
crontab -e
```
Добавьте строку (например, для синхронизации каждый час):
```
0 * * * * /usr/local/bin/sync_windows.sh
```

## 5. Решение проблем

### 5.1. Проблемы с правами
Если возникают ошибки прав доступа:
```bash
sudo rsync -avz --no-perms --no-owner --no-group --delete /path/to/source/ /mnt/windows_nfs/destination/
```

### 5.2. Проверка соединения NFS
```bash
rpcinfo -p windows_ip
showmount -e windows_ip
```

### 5.3. Альтернатива - использование SSH
Если NFS работает нестабильно, можно использовать SSH:
```bash
rsync -avz -e ssh /path/to/source/ windows_user@windows_ip:/path/to/destination/
```

## 6. Дополнительные опции

### 6.1. Ограничение скорости
```bash
rsync -avz --bwlimit=1000 --delete /path/to/source/ /mnt/windows_nfs/destination/
```

### 6.2. Исключение файлов
```bash
rsync -avz --exclude='*.tmp' --exclude='temp/' --delete /path/to/source/ /mnt/windows_nfs/destination/
```

### 6.3. Проверка изменений без синхронизации
```bash
rsync -avzn --delete /path/to/source/ /mnt/windows_nfs/destination/
```

## 7. Настройка Windows для работы с rsync (альтернативный вариант)

Если предпочтительнее использовать rsync на Windows:
1. Установите cwRsync или DeltaCopy на Windows
2. Настройте SSH-сервер на Windows (OpenSSH)
3. Используйте rsync между Linux и Windows через SSH:
```bash
rsync -avz -e ssh /path/to/source/ windows_user@windows_ip:/cygdrive/c/path/to/destination/
```