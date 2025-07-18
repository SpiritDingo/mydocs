# Создание пользователя с правами sudo в Linux

Вот несколько примеров создания пользователя с правами администратора (sudo) в Linux:

## 1. Основной способ (Ubuntu/Debian и подобные)

```bash
# Создать нового пользователя
sudo adduser username

# Добавить пользователя в группу sudo (дает права администратора)
sudo usermod -aG sudo username
```

## 2. Альтернативный способ (для CentOS/RHEL и подобных)

```bash
# Создать пользователя
sudo adduser username

# Установить пароль для пользователя
sudo passwd username

# Добавить в группу wheel (аналог sudo в RHEL-системах)
sudo usermod -aG wheel username
```

## 3. Создание пользователя с sudo правами одной командой

```bash
sudo useradd -m -G sudo username && sudo passwd username
```

## 4. Создание пользователя с SSH-ключом и sudo правами

```bash
# Создать пользователя
sudo adduser username

# Добавить в группу sudo
sudo usermod -aG sudo username

# Создать каталог .ssh и установить права
sudo mkdir -p /home/username/.ssh
sudo chmod 700 /home/username/.ssh

# Добавить публичный ключ (замените содержимое на свой ключ)
echo "ssh-rsa AAAAB3NzaC1yc2EAAA... user@example.com" | sudo tee /home/username/.ssh/authorized_keys

# Установить правильные права на файл ключа
sudo chmod 600 /home/username/.ssh/authorized_keys
sudo chown -R username:username /home/username/.ssh
```

## 5. Проверка прав sudo

После создания пользователя можно проверить его права:

```bash
sudo -l -U username
```

Эта команда покажет, какие команды пользователь может выполнять с sudo.

## Важные замечания:

1. Замените `username` на имя вашего пользователя
2. После выполнения команд пользователю нужно выйти и зайти снова, чтобы изменения вступили в силу
3. В некоторых дистрибутивах (например, Fedora) может потребоваться раскомментировать строку `%wheel ALL=(ALL) ALL` в файле `/etc/sudoers`