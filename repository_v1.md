Отличная задача! Клонирование внешнего Debian репозитория в локальную сеть — это отличный способ сэкономить интернет-трафик, ускорить установку пакетов и создать точку обновлений для изолированных сетей.

Вот несколько основных способов сделать это, от самого простого до более продвинутого.

Способ 1: apt-mirror (Простой и эффективный)

apt-mirror — это специализированная утилита, созданная именно для зеркалирования репозиториев Debian/Ubuntu.

1. Установка apt-mirror

```bash
sudo apt update
sudo apt install apt-mirror
```

2. Настройка конфигурации
Основной файл конфигурации:/etc/apt/mirror.list. Отредактируйте его.

```bash
sudo nano /etc/apt/mirror.list
```

Пример содержимого для зеркалирования части репозитория Ubuntu 22.04 Jammy:

```bash
# Базовый путь, куда будут скачиваться пакеты
set base_path /var/spool/apt-mirror

# Архитектуры (amd64, i386, arm64 и т.д.)
set defaultarch amd64

# Очищать пакеты, которые больше не в репозитории? (1=да, 0=нет)
set nthreads 20
set _tilde 0

# Ссылки на зеркала, которые будем клонировать.
# ВАЖНО: Используйте зеркало, близкое к вам географически! Проверьте список на официальном сайте.

deb http://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse
# deb http://archive.ubuntu.com/ubuntu jammy-security main restricted universe multiverse
# deb http://archive.ubuntu.com/ubuntu jammy-updates main restricted universe multiverse

# deb-src http://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse

# Для Debian это будет выглядеть так:
# deb http://deb.debian.org/debian bookworm main contrib non-free
# deb http://security.debian.org/debian-security bookworm-security main contrib non-free

clean http://archive.ubuntu.com/ubuntu
```

Закомментируйте ненужные строки (например, deb-src для исходных кодов, если они не нужны). Начните с малого — только main и updates.

3. Запуск зеркалирования

```bash
sudo apt-mirror
```

Этот процесс может занять очень много времени (часы или даже дни) в зависимости от размера выбранных репозиториев и скорости вашего интернет-канала.

4. Настройка веб-сервера для раздачи файлов
Теперь нужно сделать скачанные файлы доступными по сети.Установите, например, nginx или apache2.

```bash
sudo apt install nginx
```

Создадим симлинк из корневой директории nginx в нашу папку с зеркалом.

```bash
sudo rm -rf /var/www/html
sudo ln -s /var/spool/apt-mirror/mirror/archive.ubuntu.com/ubuntu /var/www/html
# Для Debian путь будет другим, например: /var/spool/apt-mirror/mirror/deb.debian.org/debian
```

5. Настройка клиентов для использования вашего зеркала
На других машинах в сети нужно заменить адреса репозиториев на адрес вашего сервера.

Отредактируйте файл /etc/apt/sources.list на клиентской машине:

```bash
sudo nano /etc/apt/sources.list
```

Замените все строки на подобные (замените 192.168.1.10 на IP-адрес вашего зеркал-сервера):

```
deb http://192.168.1.10/ubuntu jammy main restricted universe multiverse
deb http://192.168.1.10/ubuntu jammy-security main restricted universe multiverse
deb http://192.168.1.10/ubuntu jammy-updates main restricted universe multiverse
```

Обновите кэш:

```bash
sudo apt update
```

---

Способ 2: apt-cacher-ng (Прокси-кэш, более экономичный)

apt-cacher-ng не клонирует весь репозиторий сразу, а кэширует пакеты, которые хотя бы раз были запрошены клиентами. Это экономит место на диске.

1. Установка на сервер

```bash
sudo apt update
sudo apt install apt-cacher-ng
```

2. Настройка (опционально)
Основной конфиг:/etc/apt-cacher-ng/acng.conf. Часто достаточно настроек по умолчанию. Убедитесь, что сервер слушает на всех интерфейсах (или на нужном):

```bash
# Раскомментируйте и измените при необходимости
# BindAddress: 0.0.0.0
```

Перезапустите службу:

```bash
sudo systemctl restart apt-cacher-ng
```

3. Настройка клиентов
На клиентских машинах создайте файл,который укажет APT использовать ваш прокси.

```bash
sudo echo 'Acquire::http::Proxy "http://192.168.1.10:3142";' > /etc/apt/apt.conf.d/02proxy
```

(Замените 192.168.1.10 на IP вашего сервера с apt-cacher-ng)

Теперь все запросы apt на клиентах будут идти через ваш кэширующий сервер.

---

Способ 3: reprepro (Для создания своего собственного репозитория)

Это более сложный, но и самый гибкий способ. reprepro позволяет вам не только зеркалировать, но и управлять пакетами: добавлять свои собственные, исключать ненужные из внешних репозиториев и подписывать ваш репо своим GPG-ключом.

Этот метод лучше подходит, когда вам нужен полный контроль, а не просто полная копия чужого репозитория.

Сравнение методов

Метод Плюсы Минусы Лучше всего подходит для
apt-mirror Простая настройка, полная копия репо. Требует много места на диске, медленное первое клонирование. Сетей, где нужны все пакеты из выбранных разделов.
apt-cacher-ng Экономит место, быстро начинает работать. Не работает без интернета, не ускоряет первое скачивание пакета. Сетей с лимитным интернетом, где клиенты часто устанавливают одни и те же пакеты.
reprepro Полный контроль, можно добавлять свои пакеты. Сложная настройка и управление. Продвинутых сценариев, когда нужен кастомный набор пакетов.

Рекомендация

Для большинства задач по созданию локального зеркала в сети лучше всего начать с apt-mirror. Он делает именно то, что ожидается — создает полную, автономную копию выбранных репозиториев.

Важные замечания:

1. Место на диске: Перед началом посчитайте, сколько места займут нужные вам репозитории. Для полного зеркала Ubuntu/Debian могут потребоваться сотни гигабайт или даже терабайты.
2. Автообновление: Настройте cron-задачу (sudo crontab -e) для регулярного запуска apt-mirror (например, каждую ночь).
   ```bash
   0 3 * * * /usr/bin/apt-mirror > /var/spool/apt-mirror/var/cron.log
   ```
3. Безопасность: Убедитесь, что ваш веб-сервер (nginx/apache) доступен только из вашей локальной сети.

Отлично! Давайте подробно разберем, как зеркалировать любые DEB-репозитории в локальную сеть. Методы аналогичны, но есть важные нюансы для нестандартных репозиториев.

Основные методы зеркалирования

1. Использование apt-mirror для любых репозиториев

Установка:

```bash
sudo apt update && sudo apt install apt-mirror
```

Настройка /etc/apt/mirror.list для различных репозиториев:

Пример 1: Docker Repository

```bash
set base_path /var/spool/apt-mirror
set nthreads 20
set _tilde 0

# Docker CE Repository
deb https://download.docker.com/linux/ubuntu jammy stable
# deb-src https://download.docker.com/linux/ubuntu jammy stable

clean https://download.docker.com/linux/ubuntu
```

Пример 2: Google Chrome Repository

```bash
deb http://dl.google.com/linux/chrome/deb stable main
clean http://dl.google.com/linux/chrome/deb
```

Пример 3: PostgreSQL Repository

```bash
deb http://apt.postgresql.org/pub/repos/apt jammy-pgdg main
clean http://apt.postgresql.org/pub/repos/apt
```

Пример 4: NodeSource Repository

```bash
deb https://deb.nodesource.com/node_18.x jammy main
clean https://deb.nodesource.com/node_18.x
```

2. Универсальный метод с wget или rsync

Для репозиториев, которые не следуют стандартной структуре Debian:

```bash
# Создаем директорию для зеркала
sudo mkdir -p /var/www/mirrors
cd /var/www/mirrors

# Скачиваем всю структуру репозитория
wget --mirror --no-parent --convert-links --adjust-extension \
     --page-requisites --no-host-directories \
     https://repo.example.com/deb/

# Или используем rsync для существующих зеркал
rsync -avz rsync://repo.example.com/deb/ ./local-repo/
```

3. Создание структуры репозитория вручную

Если нужно собрать несколько пакетов в свой репозиторий:

```bash
# Установка инструментов
sudo apt install dpkg-dev reprepro

# Создаем структуру каталогов
mkdir -p /var/www/my-repo/conf

# Создаем конфигурационный файл
cat > /var/www/my-repo/conf/distributions << EOF
Origin: My Local Repository
Label: my-local-repo
Codename: focal
Architectures: amd64 i386 arm64
Components: main
Description: Local repository for custom packages
SignWith: yes
EOF

# Добавляем пакеты в репозиторий
cd /var/www/my-repo
reprepro includedeb focal /path/to/your/*.deb
```

Настройка веб-сервера для раздачи

Для Nginx:

```bash
sudo apt install nginx
sudo nano /etc/nginx/sites-available/mirrors
```

Конфигурация Nginx:

```nginx
server {
    listen 80;
    server_name mirrors.local;
    root /var/www/mirrors;
    autoindex on;
    
    location / {
        try_files $uri $uri/ =404;
    }
    
    # Для правильной работы APT с подписями
    location ~ /(Release|Release.gpg|InRelease)$ {
        add_header Cache-Control "no-store";
    }
}
```

Активируем конфиг:

```bash
sudo ln -s /etc/nginx/sites-available/mirrors /etc/nginx/sites-enabled/
sudo systemctl reload nginx
```

Автоматизация обновлений

Скрипт для автоматического зеркалирования:

```bash
#!/bin/bash
# /usr/local/bin/update-mirrors.sh

LOG_FILE="/var/log/mirror-update.log"
echo "$(date): Starting mirror update" >> $LOG_FILE

# Обновляем стандартные репозитории
apt-mirror >> $LOG_FILE 2>&1

# Специальные репозитории
wget --mirror --no-parent -np -P /var/www/mirrors \
     https://special-repo.example.com/deb/ >> $LOG_FILE 2>&1

echo "$(date): Mirror update completed" >> $LOG_FILE
```

Настройка cron для автоматического обновления:

```bash
sudo crontab -e
# Добавляем строку:
0 2 * * * /usr/local/bin/update-mirrors.sh
```

Настройка клиентских машин

Для клиентов с доступом ко всем зеркалам:

Создаем файл конфигурации:

```bash
sudo nano /etc/apt/sources.list.d/local-mirrors.list
```

Содержимое:

```bash
# Основной Ubuntu/Debian
deb http://mirror-server.local/ubuntu jammy main restricted universe multiverse
deb http://mirror-server.local/ubuntu jammy-updates main restricted universe multiverse

# Docker
deb [arch=amd64] http://mirror-server.local/download.docker.com/linux/ubuntu jammy stable

# PostgreSQL
deb http://mirror-server.local/apt.postgresql.org/pub/repos/apt jammy-pgdg main

# Свой кастомный репозиторий
deb http://mirror-server.local/my-repo focal main
```

Добавление GPG-ключей для репозиториев:

```bash
# Для Docker
curl -fsSL http://mirror-server.local/download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Для PostgreSQL
curl -fsSL http://mirror-server.local/apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | sudo apt-key add -
```

Мониторинг и обслуживание

Проверка размера зеркал:

```bash
# Размер всех зеркал
du -sh /var/spool/apt-mirror/*
du -sh /var/www/mirrors/*

# Поиск самых больших директорий
find /var/spool/apt-mirror -type f -name "*.deb" | xargs du -h | sort -hr | head -20
```

Очистка старых пакетов:

```bash
# Для apt-mirror
apt-mirror > /dev/null 2>&1

# Ручная очистка старых версий пакетов
find /var/spool/apt-mirror -name "*.deb" -mtime +30 -delete
```

Особые случаи

Зеркалирование с аутентификацией:

```bash
# Создаем файл с учетными данными
echo "user:password" > ~/.wgetrc

# Используем в скрипте
wget --mirror --user=user --password=password https://private-repo.example.com/deb/
```

Зеркалирование только определенных архитектур:

В /etc/apt/mirror.list:

```bash
set defaultarch amd64,arm64
# Исключаем i386
# set --delete-i386 no
```

Синхронизация между несколькими зеркалами:

```bash
# Используем rsync для синхронизации между серверами
rsync -avz --delete /var/www/mirrors/ backup-server:/var/www/mirrors/
```

Рекомендации для продакшн-среды

1. Резервное копирование: Настройте регулярное бэкапирование конфигурационных файлов
2. Мониторинг: Настройте алерты при сбоях обновления зеркал
3. Избыточность: Создайте несколько зеркал-серверов для балансировки нагрузки
4. Документация: Ведите список всех зазеркаленных репозиториев и их версий

Этот подход позволяет создать полноценную локальную экосистему пакетов для всей вашей сети, значительно ускоряя развертывание и обновление систем.