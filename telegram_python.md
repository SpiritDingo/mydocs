Вот Python скрипт для скачивания файлов из Telegram канала с использованием библиотеки Telethon:

```python
import asyncio
import os
from telethon import TelegramClient
from telethon.errors import SessionPasswordNeededError
from telethon.tl.types import MessageMediaDocument, MessageMediaPhoto
import configparser

# Читаем конфигурацию
config = configparser.ConfigParser()
config.read('config.ini')

# Данные для подключения (получаем в my.telegram.org)
api_id = config.get('telegram', 'api_id')
api_hash = config.get('telegram', 'api_hash')
phone = config.get('telegram', 'phone')

# Настройки
channel_username = config.get('settings', 'channel_username')  # Например: @channel_name или ссылка
download_path = config.get('settings', 'download_path', fallback='./downloads')
file_types = config.get('settings', 'file_types', fallback='all')  # all, documents, photos
limit = config.getint('settings', 'limit', fallback=0)  # 0 = все сообщения

# Создаем клиент
client = TelegramClient('session_name', api_id, api_hash)

async def download_media_from_channel():
    try:
        await client.start(phone=phone)
        print("Успешно подключились к Telegram")
        
        # Получаем информацию о канале
        try:
            channel = await client.get_entity(channel_username)
            print(f"Подключились к каналу: {channel.title}")
        except Exception as e:
            print(f"Ошибка при получении канала: {e}")
            return
        
        # Создаем папку для загрузок
        os.makedirs(download_path, exist_ok=True)
        
        # Счетчики
        total_files = 0
        downloaded_files = 0
        
        # Получаем сообщения
        messages = await client.get_messages(channel, limit=limit)
        print(f"Найдено сообщений: {len(messages)}")
        
        for message in messages:
            if message.media:
                should_download = False
                
                # Проверяем тип файла
                if file_types == 'all':
                    should_download = True
                elif file_types == 'photos' and isinstance(message.media, MessageMediaPhoto):
                    should_download = True
                elif file_types == 'documents' and isinstance(message.media, MessageMediaDocument):
                    should_download = True
                
                if should_download:
                    total_files += 1
                    try:
                        # Формируем имя файла
                        if hasattr(message.media, 'document') and message.media.document.attributes:
                            for attr in message.media.document.attributes:
                                if hasattr(attr, 'file_name') and attr.file_name:
                                    file_name = attr.file_name
                                    break
                            else:
                                file_name = f"{message.id}_{message.date.strftime('%Y%m%d_%H%M%S')}"
                        elif isinstance(message.media, MessageMediaPhoto):
                            file_name = f"photo_{message.id}_{message.date.strftime('%Y%m%d_%H%M%S')}.jpg"
                        else:
                            file_name = f"file_{message.id}_{message.date.strftime('%Y%m%d_%H%M%S')}"
                        
                        # Полный путь для сохранения
                        file_path = os.path.join(download_path, file_name)
                        
                        # Скачиваем файл
                        print(f"Скачивание: {file_name}")
                        await message.download_media(file=file_path)
                        downloaded_files += 1
                        print(f"✓ Скачано ({downloaded_files}/{total_files})")
                        
                    except Exception as e:
                        print(f"✗ Ошибка при скачивании файла {file_name}: {e}")
        
        print(f"\nЗавершено! Скачано {downloaded_files} из {total_files} файлов")
        
    except SessionPasswordNeededError:
        print("Требуется двухфакторная аутентификация")
        try:
            password = input("Введите пароль 2FA: ")
            await client.sign_in(password=password)
            await download_media_from_channel()
        except Exception as e:
            print(f"Ошибка 2FA: {e}")
    
    except Exception as e:
        print(f"Ошибка подключения: {e}")
    
    finally:
        await client.disconnect()

def create_config():
    """Создает файл конфигурации"""
    config = configparser.ConfigParser()
    
    config['telegram'] = {
        'api_id': 'YOUR_API_ID',
        'api_hash': 'YOUR_API_HASH',
        'phone': '+1234567890'
    }
    
    config['settings'] = {
        'channel_username': '@channel_name',
        'download_path': './downloads',
        'file_types': 'all',  # all, documents, photos
        'limit': '0'  # 0 = все сообщения
    }
    
    with open('config.ini', 'w', encoding='utf-8') as configfile:
        config.write(configfile)
    
    print("Файл config.ini создан. Отредактируйте его, указав свои данные.")
    print("Получите api_id и api_hash на https://my.telegram.org")

async def main():
    # Проверяем наличие конфигурации
    if not os.path.exists('config.ini'):
        create_config()
        return
    
    await download_media_from_channel()

if __name__ == '__main__':
    asyncio.run(main())
```

Файл requirements.txt

```txt
telethon>=1.34.0
cryptg>=0.4.0
```

Инструкция по использованию:

1. Установка зависимостей:

```bash
pip install -r requirements.txt
```

1. Получение API ключей:
   · Зайдите на https://my.telegram.org
   · Войдите в аккаунт
   · Создайте приложение
   · Скопируйте api_id и api_hash
2. Настройка конфигурации:
   · Запустите скрипт первый раз - он создаст config.ini
   · Отредактируйте config.ini:

```ini
[telegram]
api_id = 123456
api_hash = your_api_hash_here
phone = +71234567890

[settings]
channel_username = @channel_name
download_path = ./downloads
file_types = all
limit = 0
```

1. Запуск:

```bash
python downloader.py
```

1. При первом запуске скрипт запросит код подтверждения, который придет в Telegram.

Дополнительные возможности:

Если нужен более продвинутый функционал, вот улучшенная версия с дополнительными опциями:

```python
import asyncio
import os
import argparse
from datetime import datetime
from telethon import TelegramClient, events
from telethon.tl.types import MessageMediaDocument, MessageMediaPhoto
import configparser

class TelegramDownloader:
    def __init__(self, config_file='config.ini'):
        self.config = configparser.ConfigParser()
        self.config.read(config_file, encoding='utf-8')
        self.load_config()
        
    def load_config(self):
        # Загружаем настройки
        self.api_id = self.config.getint('telegram', 'api_id')
        self.api_hash = self.config.get('telegram', 'api_hash')
        self.phone = self.config.get('telegram', 'phone')
        self.channels = self.config.get('settings', 'channels').split(',')
        self.download_path = self.config.get('settings', 'download_path')
        self.file_types = self.config.get('settings', 'file_types')
        self.limit = self.config.getint('settings', 'limit')
        self.date_from = self.config.get('settings', 'date_from', fallback=None)
        
    async def download_from_channel(self, client, channel_name):
        """Скачивает файлы из одного канала"""
        try:
            channel = await client.get_entity(channel_name.strip())
            print(f"\nОбработка канала: {channel.title}")
            
            # Создаем папку для канала
            channel_path = os.path.join(self.download_path, 
                                       channel_name.strip().replace('@', ''))
            os.makedirs(channel_path, exist_ok=True)
            
            # Фильтры для сообщений
            filters = {}
            if self.limit > 0:
                filters['limit'] = self.limit
            if self.date_from:
                filters['offset_date'] = datetime.fromisoformat(self.date_from)
            
            messages = await client.get_messages(channel, **filters)
            
            stats = {'downloaded': 0, 'total': 0, 'skipped': 0}
            
            for message in messages:
                if not message.media:
                    continue
                    
                stats['total'] += 1
                
                # Проверяем тип файла
                if self.file_types == 'photos' and not isinstance(message.media, MessageMediaPhoto):
                    stats['skipped'] += 1
                    continue
                elif self.file_types == 'documents' and not isinstance(message.media, MessageMediaDocument):
                    stats['skipped'] += 1
                    continue
                
                # Генерируем имя файла
                file_name = self.generate_filename(message)
                file_path = os.path.join(channel_path, file_name)
                
                # Проверяем, существует ли файл
                if os.path.exists(file_path):
                    print(f"⏭ Пропуск (уже существует): {file_name}")
                    stats['skipped'] += 1
                    continue
                
                # Скачиваем
                try:
                    print(f"⬇ Скачивание: {file_name}")
                    await message.download_media(file=file_path)
                    stats['downloaded'] += 1
                    print(f"✓ Успешно: {file_name}")
                except Exception as e:
                    print(f"✗ Ошибка: {file_name} - {e}")
            
            return stats
            
        except Exception as e:
            print(f"Ошибка при обработке канала {channel_name}: {e}")
            return None
    
    def generate_filename(self, message):
        """Генерирует имя файла из сообщения"""
        date_str = message.date.strftime('%Y%m%d_%H%M%S')
        
        if hasattr(message.media, 'document'):
            # Для документов пытаемся получить оригинальное имя
            for attr in message.media.document.attributes:
                if hasattr(attr, 'file_name') and attr.file_name:
                    # Добавляем дату к имени
                    name, ext = os.path.splitext(attr.file_name)
                    return f"{date_str}_{name}{ext}"
            return f"document_{message.id}_{date_str}.bin"
        
        elif isinstance(message.media, MessageMediaPhoto):
            return f"photo_{message.id}_{date_str}.jpg"
        
        return f"file_{message.id}_{date_str}.dat"
    
    async def run_monitor(self):
        """Запускает мониторинг новых сообщений"""
        client = TelegramClient('monitor_session', self.api_id, self.api_hash)
        
        @client.on(events.NewMessage(chats=self.channels))
        async def handler(event):
            if event.message.media:
                # Автоматически скачиваем новые файлы
                channel = await event.get_chat()
                channel_name = channel.username or str(channel.id)
                channel_path = os.path.join(self.download_path, channel_name)
                os.makedirs(channel_path, exist_ok=True)
                
                file_name = self.generate_filename(event.message)
                file_path = os.path.join(channel_path, file_name)
                
                print(f"\n📥 Новый файл в {channel_name}: {file_name}")
                await event.message.download_media(file=file_path)
                print(f"✓ Скачано: {file_name}")
        
        await client.start(phone=self.phone)
        print("Мониторинг запущен. Ожидание новых сообщений...")
        await client.run_until_disconnected()
    
    async def run_download(self):
        """Запускает скачивание существующих файлов"""
        client = TelegramClient('download_session', self.api_id, self.api_hash)
        await client.start(phone=self.phone)
        
        total_stats = {'downloaded': 0, 'total': 0, 'skipped': 0}
        
        for channel in self.channels:
            stats = await self.download_from_channel(client, channel)
            if stats:
                for key in total_stats:
                    total_stats[key] += stats[key]
        
        print("\n" + "="*50)
        print("ИТОГИ:")
        print(f"Всего найдено файлов: {total_stats['total']}")
        print(f"Скачано: {total_stats['downloaded']}")
        print(f"Пропущено: {total_stats['skipped']}")
        print("="*50)
        
        await client.disconnect()

def main():
    parser = argparse.ArgumentParser(description='Скачивание файлов из Telegram каналов')
    parser.add_argument('--config', default='config.ini', help='Путь к файлу конфигурации')
    parser.add_argument('--monitor', action='store_true', help='Режим мониторинга новых сообщений')
    
    args = parser.parse_args()
    
    if not os.path.exists(args.config):
        print(f"Файл конфигурации {args.config} не найден!")
        print("Создайте config.ini на основе примера.")
        return
    
    downloader = TelegramDownloader(args.config)
    
    if args.monitor:
        asyncio.run(downloader.run_monitor())
    else:
        asyncio.run(downloader.run_download())

if __name__ == '__main__':
    main()
```

Пример config.ini для улучшенной версии:

```ini
[telegram]
api_id = 123456
api_hash = your_api_hash
phone = +71234567890

[settings]
# Список каналов через запятую
channels = @channel1, @channel2, https://t.me/channel3
# Папка для сохранения
download_path = ./telegram_downloads
# Типы файлов: all, photos, documents
file_types = all
# Лимит сообщений (0 = все)
limit = 100
# Начиная с даты (опционально)
date_from = 2024-01-01
```

Функции:

1. Скачивание из нескольких каналов
2. Фильтрация по типу файлов
3. Пропуск уже скачанных файлов
4. Организация файлов по папкам каналов
5. Добавление даты к именам файлов
6. Режим мониторинга новых сообщений
7. Статистика скачивания

Запуск:

```bash
# Обычное скачивание
python telegram_downloader.py

# Режим мониторинга
python telegram_downloader.py --monitor
```

Скрипт автоматически обработает авторизацию и сохранит сессию для последующих запусков.





Мы создадим контейнеризированное решение для скачивания файлов из Telegram каналов с использованием Docker Compose. Скрипт поддерживает как разовое скачивание, так и режим постоянного мониторинга новых сообщений.

Структура проекта

```
telegram-downloader/
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── config.env.example
├── downloader.py
└── README.md (опционально)
```

1. Python скрипт downloader.py

```python
#!/usr/bin/env python3
import os
import asyncio
import argparse
from datetime import datetime
from telethon import TelegramClient, events
from telethon.tl.types import MessageMediaDocument, MessageMediaPhoto
import logging

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

class TelegramDownloader:
    def __init__(self):
        # Чтение переменных окружения
        self.api_id = int(os.getenv('API_ID', 0))
        self.api_hash = os.getenv('API_HASH', '')
        self.phone = os.getenv('PHONE', '')
        self.channels = [ch.strip() for ch in os.getenv('CHANNELS', '').split(',') if ch.strip()]
        self.download_path = os.getenv('DOWNLOAD_PATH', '/downloads')
        self.file_types = os.getenv('FILE_TYPES', 'all')  # all, photos, documents
        self.limit = int(os.getenv('LIMIT', 0))
        self.date_from = os.getenv('DATE_FROM', None)
        self.session_file = os.getenv('SESSION_FILE', 'session/telegram.session')

        if not self.api_id or not self.api_hash or not self.phone:
            raise ValueError("API_ID, API_HASH и PHONE должны быть заданы в окружении")

    def generate_filename(self, message):
        """Генерирует имя файла на основе даты и ID сообщения"""
        date_str = message.date.strftime('%Y%m%d_%H%M%S')
        if hasattr(message.media, 'document'):
            for attr in message.media.document.attributes:
                if hasattr(attr, 'file_name') and attr.file_name:
                    name, ext = os.path.splitext(attr.file_name)
                    return f"{date_str}_{name}{ext}"
            return f"document_{message.id}_{date_str}.bin"
        elif isinstance(message.media, MessageMediaPhoto):
            return f"photo_{message.id}_{date_str}.jpg"
        return f"file_{message.id}_{date_str}.dat"

    async def download_from_channel(self, client, channel_ref):
        """Скачивает файлы из указанного канала"""
        try:
            entity = await client.get_entity(channel_ref)
            channel_name = getattr(entity, 'username', None) or str(entity.id)
            logger.info(f"Обработка канала: {channel_name}")

            # Папка для канала
            channel_path = os.path.join(self.download_path, channel_name)
            os.makedirs(channel_path, exist_ok=True)

            filters = {}
            if self.limit > 0:
                filters['limit'] = self.limit
            if self.date_from:
                filters['offset_date'] = datetime.fromisoformat(self.date_from)

            messages = await client.get_messages(entity, **filters)
            stats = {'downloaded': 0, 'skipped': 0, 'total': len(messages)}

            for msg in messages:
                if not msg.media:
                    continue

                # Фильтр по типу медиа
                if self.file_types == 'photos' and not isinstance(msg.media, MessageMediaPhoto):
                    stats['skipped'] += 1
                    continue
                if self.file_types == 'documents' and not isinstance(msg.media, MessageMediaDocument):
                    stats['skipped'] += 1
                    continue

                filename = self.generate_filename(msg)
                filepath = os.path.join(channel_path, filename)

                if os.path.exists(filepath):
                    logger.info(f"⏭ Пропуск (уже есть): {filename}")
                    stats['skipped'] += 1
                    continue

                logger.info(f"⬇ Скачивание: {filename}")
                await msg.download_media(file=filepath)
                stats['downloaded'] += 1
                logger.info(f"✓ Успешно: {filename}")

            return stats

        except Exception as e:
            logger.error(f"Ошибка при обработке канала {channel_ref}: {e}")
            return None

    async def run_download(self):
        """Разовое скачивание"""
        async with TelegramClient(self.session_file, self.api_id, self.api_hash) as client:
            await client.start(phone=self.phone)
            logger.info("Клиент Telegram запущен")

            total = {'downloaded': 0, 'skipped': 0, 'total': 0}
            for channel in self.channels:
                stats = await self.download_from_channel(client, channel)
                if stats:
                    total['downloaded'] += stats['downloaded']
                    total['skipped'] += stats['skipped']
                    total['total'] += stats['total']

            logger.info(f"Итоги: скачано {total['downloaded']}, пропущено {total['skipped']}, всего {total['total']}")

    async def run_monitor(self):
        """Режим мониторинга новых сообщений"""
        client = TelegramClient(self.session_file, self.api_id, self.api_hash)

        @client.on(events.NewMessage(chats=self.channels))
        async def handler(event):
            if not event.message.media:
                return

            # Определяем канал
            chat = await event.get_chat()
            channel_name = getattr(chat, 'username', None) or str(chat.id)
            channel_path = os.path.join(self.download_path, channel_name)
            os.makedirs(channel_path, exist_ok=True)

            filename = self.generate_filename(event.message)
            filepath = os.path.join(channel_path, filename)

            if os.path.exists(filepath):
                logger.info(f"⏭ Пропуск (уже есть): {filename}")
                return

            logger.info(f"⬇ Новый файл в {channel_name}: {filename}")
            await event.message.download_media(file=filepath)
            logger.info(f"✓ Сохранено: {filename}")

        await client.start(phone=self.phone)
        logger.info("Мониторинг запущен. Ожидание новых сообщений...")
        await client.run_until_disconnected()

def main():
    parser = argparse.ArgumentParser(description='Скачивание файлов из Telegram каналов')
    parser.add_argument('--monitor', action='store_true', help='Запустить в режиме мониторинга')
    args = parser.parse_args()

    downloader = TelegramDownloader()
    if args.monitor:
        asyncio.run(downloader.run_monitor())
    else:
        asyncio.run(downloader.run_download())

if __name__ == '__main__':
    main()
```

2. Файл зависимостей requirements.txt

```
telethon>=1.34.0
cryptg>=0.4.0
```

3. Dockerfile

```dockerfile
FROM python:3.10-slim

WORKDIR /app

# Копируем зависимости и устанавливаем их
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копируем скрипт
COPY downloader.py .

# Создаем директории для сессии и загрузок
RUN mkdir -p /app/session /downloads

# Точка входа: запускаем скрипт с переданными аргументами
ENTRYPOINT ["python", "/app/downloader.py"]
```

4. docker-compose.yml

```yaml
version: '3.8'

services:
  # Сервис для разового скачивания
  download:
    build: .
    container_name: tg_downloader_once
    env_file:
      - config.env
    volumes:
      - ./downloads:/downloads
      - ./session:/app/session
    command: []  # без --monitor – разовое скачивание
    profiles:
      - once

  # Сервис для постоянного мониторинга
  monitor:
    build: .
    container_name: tg_downloader_monitor
    restart: unless-stopped
    env_file:
      - config.env
    volumes:
      - ./downloads:/downloads
      - ./session:/app/session
    command: ["--monitor"]
    profiles:
      - monitor
```

5. Пример файла переменных окружения config.env.example

```
# Получить на https://my.telegram.org
API_ID=123456
API_HASH=your_api_hash_here
PHONE=+71234567890

# Каналы через запятую (можно username или ссылки)
CHANNELS=@channel1,@channel2,https://t.me/channel3

# Папка для загрузок (внутри контейнера)
DOWNLOAD_PATH=/downloads

# Типы файлов: all, photos, documents
FILE_TYPES=all

# Лимит сообщений (0 = без лимита)
LIMIT=100

# Начиная с даты (опционально, формат YYYY-MM-DD)
DATE_FROM=2024-01-01

# Файл сессии (путь внутри контейнера)
SESSION_FILE=/app/session/telegram.session
```

Инструкция по запуску

1. Подготовка

Скопируйте файлы проекта в отдельную папку. Переименуйте config.env.example в config.env и отредактируйте его, указав свои данные (API_ID, API_HASH, PHONE, список каналов).

2. Первый запуск для создания сессии

При первом запуске Telegram запросит код подтверждения. Для этого нужно запустить контейнер в интерактивном режиме. Выполните:

```bash
docker-compose run --rm download
```

Введите запрошенный код. После успешной авторизации файл сессии сохранится в папку ./session на хосте (благодаря монтированию тома). Теперь сессия готова для дальнейшего использования.

3. Разовое скачивание

Чтобы скачать файлы из каналов один раз, используйте профиль once:

```bash
docker-compose --profile once up
```

Контейнер выполнит задачу и остановится.

4. Постоянный мониторинг

Для запуска в режиме мониторинга новых сообщений:

```bash
docker-compose --profile monitor up -d
```

Контейнер будет работать в фоне, автоматически перезапускаться при сбоях (restart: unless-stopped) и скачивать все новые файлы из указанных каналов.

5. Остановка мониторинга

```bash
docker-compose --profile monitor down
```

6. Просмотр логов

```bash
docker-compose --profile monitor logs -f
```

Важные замечания

· Безопасность: никогда не коммитьте config.env и папку session в Git – добавьте их в .gitignore.
· Двухфакторная аутентификация: если у вас включена 2FA, при первом запуске потребуется ввести пароль. Скрипт поддерживает это.
· Обновление списка каналов: после изменения CHANNELS в config.env перезапустите контейнер мониторинга.
· Права доступа: убедитесь, что папки downloads и session доступны для записи контейнеру (обычно достаточно chmod 777 или запуска от текущего пользователя).

Заключение

Вы получили полностью контейнеризированное решение для скачивания файлов из Telegram каналов. Оно легко разворачивается, масштабируется и может работать в фоне, автоматически сохраняя новые медиафайлы.