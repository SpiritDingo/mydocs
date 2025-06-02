### **Rsync между Linux и Windows: NFS, SMB (Samba), SSH**  

В этом руководстве рассмотрим **3 способа** синхронизации файлов между Linux и Windows:  
1. **Rsync + SSH** (если на Windows установлен OpenSSH)  
2. **Rsync + SMB (Samba)** (если Windows расшаривает папку)  
3. **Rsync + NFS** (если Windows настроен как NFS-сервер)  

---

## **① Способ 1: Rsync через SSH (Windows как сервер)**  
### **Требования:**  
- На Windows установлен **SSH-сервер** (OpenSSH).  
- Rsync на Linux (обычно предустановлен).  

### **1. Настройка OpenSSH на Windows**  
1. **Windows 10/11:**  
   ```powershell
   # Установка OpenSSH (если нет)
   Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
   # Запуск и добавление в автозагрузку
   Start-Service sshd
   Set-Service -Name sshd -StartupType 'Automatic'
   ```
2. **Проверка подключения с Linux:**  
   ```bash
   ssh username@windows_ip
   ```

### **2. Синхронизация через rsync**  
```bash
rsync -avz --progress -e "ssh -p 22" /путь/на/linux/ username@windows_ip:/C:/путь/на/windows/
```
- **`/C:/`** — указывает на диск `C:` в Windows.  
- Если SSH на нестандартном порту → `-p 2222`.  

---

## **② Способ 2: Rsync через SMB (Samba, общая папка Windows)**  
### **Требования:**  
- Windows расшаривает папку (SMB).  
- Linux может монтировать SMB-шару (`cifs-utils`).  

### **1. Настройка общей папки на Windows**  
1. Правой кнопкой → **Свойства → Доступ → Общий доступ**.  
2. Дайте права (например, `Everyone` **Чтение/Запись**).  
3. Запомните путь: `\\IP\ShareName`.  

### **2. Монтирование SMB на Linux**  
```bash
sudo apt install cifs-utils  # Debian/Ubuntu
sudo yum install cifs-utils  # CentOS/RHEL

sudo mkdir /mnt/win_share
sudo mount -t cifs -o username=win_user,password=пароль //windows_ip/ShareName /mnt/win_share
```
- Если пароль пустой → `password=""`.  
- Если доменная учётка → `username=DOMAIN\User`.  

### **3. Rsync между Linux и смонтированной папкой**  
```bash
rsync -avz --progress /путь/на/linux/ /mnt/win_share/
```
- После синхронизации отмонтируйте:  
  ```bash
  sudo umount /mnt/win_share
  ```

---

## **③ Способ 3: Rsync через NFS (если Windows — NFS-сервер)**  
### **Требования:**  
- Windows поддерживает **NFS-сервер** (только **Pro/Enterprise** версии).  
- Linux может монтировать NFS (`nfs-common`).  

### **1. Включение NFS-сервера на Windows**  
1. **Панель управления → Программы → Включение компонентов Windows**.  
2. Выберите:  
   - **Служба NFS → Сервер NFS**  
   - **Клиент NFS** (если нужно и с Windows на Linux).  

### **2. Экспорт папки на Windows**  
1. Правой кнопкой → **Свойства → NFS-доступ**.  
2. Настройте права (например, `Read/Write` для Linux-IP).  

### **3. Монтирование NFS на Linux**  
```bash
sudo apt install nfs-common  # Debian/Ubuntu
sudo yum install nfs-utils   # CentOS/RHEL

sudo mkdir /mnt/win_nfs
sudo mount -t nfs windows_ip:/C/путь/на/windows /mnt/win_nfs
```
- Если нужны особые права:  
  ```bash
  sudo mount -t nfs -o rw,noatime windows_ip:/Share /mnt/win_nfs
  ```

### **4. Rsync через NFS**  
```bash
rsync -avz --progress /путь/на/linux/ /mnt/win_nfs/
```
- После работы отмонтируйте:  
  ```bash
  sudo umount /mnt/win_nfs
  ```

---

## **Сравнение методов**  
| Метод       | Скорость | Безопасность | Простота настройки |  
|-------------|---------|-------------|-------------------|  
| **SSH**     | ⚡⚡⚡ | 🔒🔒🔒 (шифрование) | Средняя (настройка SSH) |  
| **SMB**     | ⚡⚡     | 🔒 (пароль в plaintext) | Проще (если есть общая папка) |  
| **NFS**     | ⚡⚡⚡   | 🔒 (без шифрования) | Сложнее (только Windows Pro) |  

---

## **Автоматизация (cron + rsync)**  
Добавьте в `crontab -e`:  
```bash
# Rsync каждые 2 часа через SSH
0 */2 * * * rsync -avz -e "ssh -p 22" /home/user/docs/ win_user@192.168.1.100:/C:/Backup/
```
- Логи: `>> /var/log/rsync.log 2>&1`.  

---

## **Вывод**  
- **Лучшая безопасность** → **SSH**.  
- **Простота** → **SMB** (если уже есть общая папка).  
- **Максимальная скорость** → **NFS** (но сложнее в настройке).  

**Пример полной команды (SMB):**  
```bash
rsync -avz --delete --progress /data/ /mnt/win_share/backup/
```
(Удалит лишние файлы в `backup/` и покажет прогресс.)  

Нужны уточнения? Задавайте вопросы! 🚀