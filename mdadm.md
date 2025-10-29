Если RAID массив "развалился" (перестал работать после перезагрузки или сбоя), вот как его правильно собрать заново:

1. Определение компонентов RAID

Сначала найдем диски, которые входили в массив:

```bash
# Просмотр всех RAID-устройств в системе
cat /proc/mdstat

# Поиск дисков с RAID-метаданными
sudo mdadm --examine /dev/sd*
# или конкретно по дискам
sudo mdadm --examine /dev/sdb /dev/sdc /dev/sdd

# Детальная информация о найденных RAID-компонентах
sudo mdadm --examine --scan
```

2. Сборка существующего массива

Если массив просто размонтирован, но диски на месте:

```bash
# Автоматическая сборка всех найденных массивов
sudo mdadm --assemble --scan

# Ручная сборка конкретного массива
sudo mdadm --assemble /dev/md0 /dev/sdb1 /dev/sdc1 /dev/sdd1

# Если не знаем точный состав - попробовать автоматически
sudo mdadm --assemble /dev/md0 --uuid=UUID_массива
```

3. Если массив не собирается автоматически

Проверка состояния дисков:

```bash
# Проверим, нет ли поврежденных дисков
sudo badblocks -v /dev/sdb
sudo smartctl -a /dev/sdb

# Смотрим метаданные на каждом диске
sudo mdadm --examine /dev/sdb
sudo mdadm --examine /dev/sdc
```

Принудительная сборка (если нужно):

```bash
# Если некоторые диски помечены как faulty
sudo mdadm --assemble /dev/md0 /dev/sdb /dev/sdc --force

# Если метаданные повреждены
sudo mdadm --assemble /dev/md0 /dev/sdb /dev/sdc --run
```

4. Восстановление после потери диска

Если один из диков вышел из строя:

```bash
# Собрать массив в деградированном состоянии
sudo mdadm --assemble /dev/md0 /dev/sdb /dev/sdc  # без вышедшего диска

# Добавить новый диск для замены
sudo mdadm --add /dev/md0 /dev/sdd

# Дождаться завершения синхронизации
cat /proc/mdstat
```

5. Если конфигурация потеряна

Воссоздаем массив на основе существующих данных:

```bash
# Создаем массив в режиме assemble
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc --assume-clean
```

Внимание! Будьте осторожны с --assume-clean - используйте только если уверены в целостности данных.

6. Восстановление суперблока

Если суперблоки повреждены:

```bash
# Резервное копирование суперблоков
sudo mdadm --examine /dev/sdb > /root/sdb_superblock.backup

# Восстановление суперблока
sudo mdadm --zero-superblock /dev/sdb  # Осторожно!
sudo mdadm --add /dev/md0 /dev/sdb
```

7. Пошаговый пример восстановления

Предположим, был RAID1 из /dev/sdb и /dev/sdc:

```bash
# 1. Проверяем диски
sudo mdadm --examine /dev/sdb /dev/sdc

# 2. Пробуем собрать
sudo mdadm --assemble /dev/md0 /dev/sdb /dev/sdc

# 3. Если не работает, смотрим ошибку
dmesg | grep md

# 4. Пробуем принудительно
sudo mdadm --assemble /dev/md0 /dev/sdb /dev/sdc --force

# 5. Проверяем состояние
cat /proc/mdstat
sudo mdadm --detail /dev/md0

# 6. Монтируем
sudo mount /dev/md0 /mnt/raid
```

8. Критические моменты

· Не используйте --create без --assume-clean - это уничтожит данные!
· Всегда делайте backup перед манипуляциями
· Проверяйте журналы: dmesg | grep md
· Если данные критичны - обратитесь к специалистам

9. После успешного восстановления

```bash
# Сохраняем конфигурацию
sudo mdadm --detail --scan >> /etc/mdadm/mdadm.conf

# Обновляем initramfs
sudo update-initramfs -u

# Добавляем в fstab для автозагрузки
echo "/dev/md0 /mnt/raid ext4 defaults 0 0" >> /etc/fstab
```

Помните: если RAID развалился из-за аппаратных проблем, сначала решите их (замените диски, проверьте кабели), и только потом восстанавливайте массив.



Приведу конкретные примеры восстановления RAID после сбоя.

Пример 1: RAID1 перестал монтироваться после перезагрузки

Ситуация:

Был RAID1 из /dev/sdb и /dev/sdc, после перезагрузки массив не собран.

Диагностика:

```bash
# Смотрим состояние RAID
cat /proc/mdstat
# Вывод: нет активных массивов

# Проверяем диски
lsblk
# Вывод:
# sdb   8:16   0   100G  0 disk
# sdc   8:32   0   100G  0 disk

# Смотрим метаданные на дисках
sudo mdadm --examine /dev/sdb
# Вывод:
# /dev/sdb:
#           Magic : a92b4efc
#         Version : 1.2
#     Feature Map : 0.0
#      Array UUID : a1b2c3d4:e5f6a7b8:c9d0e1f2:12345678
#            Name : server:0
#   Creation Time : Fri Oct 15 10:30:00 2023
#      Raid Level : raid1

sudo mdadm --examine /dev/sdc
# Аналогичные метаданные
```

Восстановление:

```bash
# Собираем массив
sudo mdadm --assemble /dev/md0 /dev/sdb /dev/sdc
# Вывод: mdadm: /dev/md0 has been started with 2 drives.

# Проверяем
cat /proc/mdstat
# Вывод:
# Personalities : [raid1]
# md0 : active raid1 sdc[1] sdb[0]
#       104755200 blocks super 1.2 [2/2] [UU]

# Смотрим детали
sudo mdadm --detail /dev/md0
# Вывод показывает, что массив собран и работает

# Монтируем
sudo mount /dev/md0 /mnt/raid
ls -la /mnt/raid
# Видим наши файлы - всё работает!
```

---

Пример 2: RAID5 с одним отказавшим диском

Ситуация:

RAID5 из 3 дисков (/dev/sdb, /dev/sdc, /dev/sdd), один диск вышел из строя.

Диагностика:

```bash
# Пробуем собрать
sudo mdadm --assemble /dev/md0 /dev/sdb /dev/sdc /dev/sdd
# Вывод: mdadm: /dev/md0 assembled from 2 drives - not enough to start the array.

# Смотрим детали дисков
sudo mdadm --examine /dev/sdb /dev/sdc /dev/sdd
# /dev/sdd показывает ошибки или не отвечает

# Проверяем физическое состояние
sudo smartctl -a /dev/sdd
# Вывод: Read SMART Error Log failed: scsi error badly formed scsi parameters
# Диск sdd неисправен
```

Восстановление:

```bash
# Собираем в деградированном состоянии
sudo mdadm --assemble /dev/md0 /dev/sdb /dev/sdc --force
# Вывод: mdadm: /dev/md0 has been started with 2 drives (out of 3).

# Проверяем состояние
cat /proc/mdstat
# Вывод:
# md0 : active raid5 sdc[2] sdb[1]
#       209510400 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [_UU]

sudo mdadm --detail /dev/md0
# Вывод показывает состояние "degraded"

# Монтируем в режиме только для чтения (на всякий случай)
sudo mount -o ro /dev/md0 /mnt/raid

# Заменяем неисправный диск
# Предположим, новый диск - /dev/sde
sudo mdadm --add /dev/md0 /dev/sde
# Вывод: mdadm: added /dev/sde

# Наблюдаем за восстановлением
watch cat /proc/mdstat
# Вывод будет показывать прогресс синхронизации:
# md0 : active raid5 sde[3] sdc[2] sdb[1]
#       209510400 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [_UU]
#       [=>...................]  recovery =  8.3% (17432576/209510400) finish=20.1min speed=159100K/sec

# После завершения восстановления
sudo mdadm --detail /dev/md0
# Вывод показывает [3/3] [UUU] - массив восстановлен
```

---

Пример 3: RAID1 с конфликтом метаданных

Ситуация:

После неудачной попытки создания массива метаданные повреждены.

Диагностика:

```bash
sudo mdadm --examine /dev/sdb
# Вывод: mdadm: No md superblock detected on /dev/sdb

# Или может быть старая версия метаданных
```

Восстановление:

```bash
# Очищаем старые метаданные (ОСТОРОЖНО!)
sudo mdadm --zero-superblock /dev/sdb
sudo mdadm --zero-superblock /dev/sdc

# Создаем новый массив
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
# Вывод: mdadm: Defaulting to version 1.2 metadata
# mdadm: array /dev/md0 started.

# Ждем синхронизации
watch cat /proc/mdstat

# Создаем файловую систему
sudo mkfs.ext4 /dev/md0

# Монтируем
sudo mount /dev/md0 /mnt/raid
```

---

Пример 4: Восстановление после полного сбоя системы

Ситуация:

После замены материнской платы RAID не определяется.

Восстановление:

```bash
# Сканируем все диски
sudo mdadm --assemble --scan
# Вывод: mdadm: No arrays found in config file or automatically

# Ищем RAID компоненты на всех дисках
for disk in /dev/sd[b-z]; do
    if [ -e $disk ]; then
        echo "=== $disk ==="
        sudo mdadm --examine $disk 2>/dev/null | grep -E "UUID|Level|Devices"
    fi
done

# Вывод:
# === /dev/sdb ===
#    UUID : a1b2c3d4:e5f6a7b8:c9d0e1f2:12345678
#    Raid Level : raid1
#    Raid Devices : 2
# === /dev/sdc ===
#    UUID : a1b2c3d4:e5f6a7b8:c9d0e1f2:12345678
#    Raid Level : raid1
#    Raid Devices : 2

# Собираем по UUID
sudo mdadm --assemble /dev/md0 --uuid=a1b2c3d4:e5f6a7b8:c9d0e1f2:12345678
# Вывод: mdadm: /dev/md0 has been started with 2 drives.

# Сохраняем конфигурацию
sudo mdadm --detail --scan | sudo tee /etc/mdadm/mdadm.conf

# Обновляем initramfs
sudo update-initramfs -u
```

---

Пример 5: Восстановление с одним рабочим диском RAID1

Ситуация:

Один диск полностью вышел из строя, второй рабочий.

```bash
# Создаем новый массив на основе рабочего диска
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb missing --assume-clean
# Вывод: mdadm: /dev/md0 has been started with 1 drive (out of 2).

# Проверяем, что данные доступны
sudo mount /dev/md0 /mnt/raid
ls -la /mnt/raid
# Видим файлы - данные целы

# Добавляем новый диск для замены
sudo mdadm --add /dev/md0 /dev/sdc

# Ждем синхронизации
watch cat /proc/mdstat
```

---

Полезные команды для мониторинга:

```bash
# Постоянный мониторинг восстановления
watch -n 1 'cat /proc/mdstat; echo; df -h /mnt/raid'

# Детальный прогресс
cat /sys/block/md0/md/sync_action
cat /sys/block/md0/md/sync_completed

# Проверка целостности после восстановления
sudo mdadm --detail /dev/md0 | grep -i consistency
```

Важно: Всегда делайте бэкап критичных данных перед манипуляциями с RAID!