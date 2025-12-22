Вот несколько способов запуска gzip-сжатия в несколько потоков в Linux:

1. pigz (Parallel gzip) - самый популярный вариант

```bash
# Установка
sudo apt install pigz    # Debian/Ubuntu
sudo yum install pigz    # CentOS/RHEL
sudo brew install pigz   # macOS

# Использование
pigz file.txt            # сжатие с использованием всех ядер
pigz -k file.txt         # сжатие с сохранением оригинала (-k)
pigz -p 4 file.txt       # указать количество потоков
pigz -d file.txt.gz      # распаковка

# Сжатие с tar
tar -cf - directory/ | pigz > archive.tar.gz
tar --use-compress-program=pigz -cf archive.tar.gz directory/
```

2. gzip с GNU Parallel

```bash
# Установка parallel
sudo apt install parallel

# Сжатие нескольких файлов параллельно
find . -name "*.log" -type f | parallel -j 4 gzip

# Сжатие одного большого файла блоками
cat bigfile.txt | parallel --pipe --block 10M gzip > compressed.gz
```

3. Многопоточное сжатие при архивировании

```bash
# С tar и pigz
tar -I pigz -cf archive.tar.gz /path/to/dir

# С использованием pbzip2 (для bzip2)
tar -I pbzip2 -cf archive.tar.bz2 /path/to/dir

# Современный zstd с многопоточностью
tar -I 'zstd -T0' -cf archive.tar.zst /path/to/dir
```

4. Параллельная обработка нескольких файлов

```bash
# Используя xargs
find . -name "*.txt" -type f | xargs -P 4 -I {} gzip {}

# Используя parallel с сохранением структуры
find . -name "*.csv" -type f | parallel --eta gzip
```

5. Скрипт для параллельного сжатия

```bash
#!/bin/bash
# compress_parallel.sh
THREADS=4
DIR="/path/to/files"

compress_file() {
    local file=$1
    echo "Сжатие $file"
    gzip "$file"
}

export -f compress_file

find "$DIR" -name "*.log" -type f | parallel -j $THREADS compress_file
```

6. Сравнение производительности

Утилита Скорость Сжатие Потоки
gzip 1x Хорошо 1
pigz 4-8x Так же Много
zstd 10-15x Лучше Много

Примеры использования:

Сжатие логов за последние 7 дней:

```bash
find /var/log -name "*.log" -mtime -7 -type f | parallel -j $(nproc) gzip
```

Распараллеленное создание бэкапа:

```bash
# Создание архива с использованием всех ядер
tar -cf - /important/data | pigz -p $(nproc) > backup_$(date +%Y%m%d).tar.gz

# С прогресс-баром (если установлен pv)
tar -cf - /data | pv | pigz -p 8 > backup.tar.gz
```

Мониторинг процесса:

```bash
# Просмотр использования CPU при сжатии
watch -n 1 'ps aux | grep -E "(gzip|pigz)" | grep -v grep'
```

Важные замечания:

1. pigz обеспечивает полную совместимость с gzip
2. При сжатии многих маленьких файлов учитывайте нагрузку на диск
3. Для максимальной скорости используйте флаг -1 (наименьшее сжатие)
4. Для лучшего сжатия используйте -9, но это медленнее

```bash
# Быстрое сжатие
pigz -1 -p 8 large_file.iso

# Максимальное сжатие
pigz -9 -p 4 important_document.db
```

Выбор инструмента зависит от задачи: для полной совместимости используйте pigz, для максимальной скорости - zstd.