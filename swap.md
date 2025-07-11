Чтобы добавить swap-файл в Ubuntu, выполните следующие шаги:

### 1. Проверьте текущий swap
```bash
free -h
swapon --show
```
Если в выводе нет активного swap, продолжите.

---

### 2. Создайте swap-файл
Замените `2G` на нужный размер (например, `1G`, `4G`).  
**Рекомендации**:  
- Для RAM ≤ 2 GB: размер swap = 2 × RAM.  
- Для RAM > 2 GB: размер swap = RAM.

```bash
sudo fallocate -l 2G /swapfile
```
Если `fallocate` не работает, используйте:
```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048
```

---

### 3. Настройте права доступа
```bash
sudo chmod 600 /swapfile
```

---

### 4. Форматируйте файл под swap
```bash
sudo mkswap /swapfile
```

---

### 5. Активируйте swap
```bash
sudo swapon /swapfile
```

---

### 6. Сделайте изменение постоянным
Добавьте запись в `/etc/fstab`:
```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

### 7. Проверьте результат
```bash
free -h
swapon --show
```

---

### Опционально: Настройка swappiness
**Swappiness** (частота использования swap, 0-100).  
Проверьте текущее значение:
```bash
cat /proc/sys/vm/swappiness
```
Измените временно (сбросится после перезагрузки):
```bash
sudo sysctl vm.swappiness=10
```
Для постоянного изменения:
```bash
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

### Если нужно удалить swap:
```bash
sudo swapoff /swapfile
sudo rm /swapfile
sudo sed -i '/\/swapfile/d' /etc/fstab
```

### Важные замечания:
1. Размер swap не должен превышать доступное место на диске.
2. Использование SSD: частые операции с swap могут сократить срок службы диска.
3. Для серверов рекомендуется выделять отдельный раздел под swap.