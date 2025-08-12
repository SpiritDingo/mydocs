### **Подробная пошаговая инструкция: Настройка Sonatype Nexus для хранения шрифтов `ttf-mscorefonts-installer` (например, `andale32.exe`)**  

В этом руководстве мы:  
1. **Создадим Proxy-репозиторий** для загрузки файлов с SourceForge.  
2. **Создадим Hosted Raw-репозиторий** (если нужно локальное хранилище).  
3. **Настроим Debian/Ubuntu** на использование Nexus вместо SourceForge.  

---

## **1. Настройка Nexus Repository Manager**  

### **1.1. Создание Proxy-репозитория для SourceForge**  
(Если файлы должны скачиваться напрямую с SourceForge, но кэшироваться в Nexus)  

1. **Зайдите в Nexus** (`http://ваш-сервер:8081`) → **Sign In** (логин/пароль по умолчанию `admin` и пароль из `sonatype-work/nexus3/admin.password`).  
2. Перейдите:  
   - **⚙ Settings (шестерёнка)** → **Repository** → **Repositories** → **Create repository**.  
3. Выберите **Proxy repository** (чтобы Nexus кэшировал файлы с удалённого сервера).  
4. Заполните настройки:  
   - **Name**: `sourceforge-corefonts`  
   - **Remote URL**: `http://downloads.sourceforge.net/`  
   - **Blob store**: `default`  
   - **Version policy**: `Release` (или `Mixed`)  
   - **Layout policy**: `Permissive` (важно, так как SourceForge не использует Maven-структуру)  
   - **HTTP Authentication**: Оставьте пустым (если не требуется).  
5. Нажмите **Create repository**.  

### **1.2. (Опционально) Создание Hosted Raw-репозитория**  
(Если нужно вручную загрузить `.exe`-файлы в Nexus, например, для локального хранения)  

1. **Repositories** → **Create repository** → **Raw (hosted)**.  
2. Настройки:  
   - **Name**: `corefonts-local`  
   - **Blob store**: `default`  
   - **Version policy**: `Release`  
   - **Deployment policy**: `Allow redeploy` (если нужно перезаписывать файлы).  
3. Нажмите **Create repository**.  

### **1.3. Загрузка файлов в Hosted-репозиторий (если выбран этот вариант)**  
Вы можете загрузить файлы вручную:  
- Через веб-интерфейс:  
  - Откройте `corefonts-local` → **Upload** → перетащите `andale32.exe`.  
- Через `curl`:  
  ```bash
  curl -u admin:ваш_пароль --upload-file andale32.exe \
    http://ваш-nexus:8081/repository/corefonts-local/corefonts/andale32.exe
  ```

---

## **2. Настройка `ttf-mscorefonts-installer` для работы с Nexus**  

### **2.1. Вариант 1: Proxy-репозиторий (автоматическая загрузка из SourceForge через Nexus)**  
1. Узнайте URL файла в Nexus (например):  
   ```
   http://ваш-nexus:8081/repository/sourceforge-corefonts/corefonts/andale32.exe
   ```
2. Настройте `ttf-mscorefonts-installer` на использование этого URL:  
   ```bash
   sudo debconf-set-selections <<< "ttf-mscorefonts-installer msttcorefonts/dlurl string http://ваш-nexus:8081/repository/sourceforge-corefonts/corefonts/"
   ```
3. Переустановите пакет:  
   ```bash
   sudo apt purge ttf-mscorefonts-installer
   sudo apt install ttf-mscorefonts-installer
   ```

### **2.2. Вариант 2: Hosted-репозиторий (локальное хранилище в Nexus)**  
1. Убедитесь, что файлы загружены (например, `andale32.exe` лежит в `corefonts-local`).  
2. Настройте пакет на использование локального URL:  
   ```bash
   sudo debconf-set-selections <<< "ttf-mscorefonts-installer msttcorefonts/dlurl string http://ваш-nexus:8081/repository/corefonts-local/corefonts/"
   ```
3. Переустановите:  
   ```bash
   sudo apt purge ttf-mscorefonts-installer
   sudo apt install ttf-mscorefonts-installer
   ```

---

## **3. Проверка работы**  
1. При установке `ttf-mscorefonts-installer` пакет должен брать файлы из Nexus.  
2. Проверьте логи Nexus (`sonatype-work/nexus3/log/request.log`), чтобы убедиться, что файлы скачиваются.  
3. Если файлы не загружаются, проверьте:  
   - Доступность URL (`curl -v http://nexus:8081/.../andale32.exe`).  
   - Настройки `debconf` (`sudo debconf-show ttf-mscorefonts-installer`).  

---

## **Итог**  
- **Proxy-репозиторий** → автоматически кэширует файлы с SourceForge.  
- **Hosted Raw-репозиторий** → локальное хранилище (если нужен полный контроль).  
- Настройка `ttf-mscorefonts-installer` через `debconf` позволяет перенаправить загрузку на Nexus.  

Теперь шрифты будут скачиваться из вашего Nexus, а не напрямую из интернета! 🚀