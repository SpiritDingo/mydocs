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