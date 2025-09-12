# Статья на главной странице index.html

## Инструкция по настройке и запуску веб-сервера (AlmaLinux 8)

**Внимание:** инструкция адаптирована для AlmaLinux 8.

### 1. Настройка записей DNS

Добавьте **A**-запись с IPv4-адресом сервера для **your_domain.com** и для **www.your_domain.com**. Для IPv6 — **AAAA**-запись.

**Важно:** DNS-записи должны быть настроены ДО получения SSL-сертификата, так как процесс проверки права на домен требует доступности вашего домена.

---

### 2. Создаем пользователя с правами sudo

- Добавить пользователя:
```bash
sudo useradd your_username
```

- Задать пароль:
```bash
sudo passwd your_username
```

- Добавить в группу wheel:
```bash
sudo usermod -aG wheel your_username
```

---

### 3. Настройка SSH

**Заметка:** Для настройки ключей в Windows 11 смотрите: [Настройка OpenSSH в Windows 11](pages/ssh-keys-windows.html)

#### Генерация и копирование ключей

- Генерация ключей на клиенте:
```bash
ssh-keygen
```

- Копирование публичного ключа на сервер:
```bash
ssh-copy-id user@host
```
*Если ssh-copy-id недоступен, скопируйте `~/.ssh/id_rsa.pub` в `~/.ssh/authorized_keys` на сервере.*

#### Редактирование конфигурации sshd

**Внимание:** перед отключением паролей проверьте подключение по ключу в новой сессии.

Откройте файл конфигурации и внесите изменения:
```bash
sudo nano /etc/ssh/sshd_config
```

- Отключить парольную аутентификацию:
```
PasswordAuthentication no
```

- Запретить вход root:
```
PermitRootLogin no
```

- Изменить порт (пример):
```
Port 2222
```

Применить изменения:
```bash
sudo systemctl reload sshd
```

#### SeLinux (разрешение нового порта)

**Заметка:** Если SeLinux отключен, пропускаем этот шаг.

- Проверить статус SeLinux:
```bash
sestatus
```

- Установить утилиты (AlmaLinux 8):
```bash
sudo dnf install -y policycoreutils-python-utils
```

- Применить политику SeLinux к порту 2222 (назначить тип ssh_port_t):
```bash
sudo semanage port -a -t ssh_port_t -p tcp 2222
```

- Проверить:
```bash
semanage port -l | grep ssh
```

#### FirewallD (открытие порта)

- Открыть порт:
```bash
sudo firewall-cmd --permanent --add-port=2222/tcp
```

- Перезагрузим FirewallD:
```bash
sudo firewall-cmd --reload
```

- Проверка:
```bash
sudo firewall-cmd --list-ports
sudo firewall-cmd --query-port=2222/tcp
```

#### Права на ключи и тест подключения

Если ключ создан командой `ssh-keygen`, нужные права будут установлены автоматически.

- Установить права:
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/*
```

- Тестирование:
```bash
ssh -p 2222 user@host
```
*Не закрывайте старую сессию до успешного теста.*

Начальная настройка SSH завершена.

---

### 4. Установка и запуск веб-сервера

Список действий:
1. Установить веб-сервер (Apache или Nginx)
2. Открыть порт в firewalld
3. Включить автозапуск службы

#### Установка (AlmaLinux 8)

- Apache:
```bash
sudo dnf install -y httpd
sudo systemctl enable --now httpd
```

- Nginx:
```bash
sudo dnf install -y nginx
sudo systemctl enable --now nginx
```

#### Открытие портов для HTTP и HTTPS

- Добавить сервис HTTP и HTTPS:  
**Примечание**: В системе AlmaLinux 8, при установке веб-сервера Apache, порты автоматически добавились в открытые. Добавлять ничего не надо, нужно только
проверить *конфигурацию зоны*, чтобы убедиться что порты открыты.
```bash
sudo firewall-cmd --zone=public --permanent --add-service=http
sudo firewall-cmd --zone=public --permanent --add-service=https
sudo firewall-cmd --reload
```

- Проверить конфигурацию зоны:
```bash
sudo firewall-cmd --zone=public --list-all
```

- Проверить доступность:
```bash
curl -I http://YOUR_SERVER_IP
```

---

### 5. Получение и настройка SSL-сертификата Let`s Encrypt

- [Документация Let`s Encrypt](https://letsencrypt.org/ru/docs/)
- [Документация certbot](https://certbot.eff.org/)

Инструкция исходит из анализа, который представлен в моём посте [О сертификатах Let`s Encrypt](pages/ssh-keys-windows.html).

#### Установка Certbot

Certbot — это официальный клиент для получения и управления сертификатами Let's Encrypt.

- Установите EPEL-репозиторий и Certbot:
```bash
sudo dnf install -y epel-release
sudo dnf install -y certbot python3-certbot-nginx   # для Nginx
# или
sudo dnf install -y certbot python3-certbot-apache  # для Apache
```

#### Автоматическая настройка для Apache

- Делаем бекап. Переименовываем конфиг `/etc/httpd/conf.d/ssl.conf`  
этот файл конфигурации создается автоматически в AlmaLinux 8, при установке httpd.  В нём подключается сертификат  
`/etc/pki/tls/certs/localhost.crt` для локальной разработке.
```bash
sudo mv /etc/httpd/conf.d/ssl.conf /etc/httpd/conf.d/ssl.conf.bak
```
- Создаем виртуальный хост
```
sudo vim /etc/httpd/conf.d/linuxgik.ru.conf
```

```apache
<VirtualHost *:80>
    ServerName linuxgik.ru
    ServerAlias www.linuxgik.ru
    DocumentRoot /var/www/html
    
    ErrorLog /var/log/httpd/linuxgik.ru-error.log
    CustomLog /var/log/httpd/linuxgik.ru-access.log combined
    
    <Directory "/var/www/html">
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Проверяем синтаксис:
```bash
sudo apachectl configtest
```
Перезагружаем:
```bash
sudo systemctl reload httpd
```

Проверяем что виртуальный хост работает
```bash
curl -I http://linuxgik.ru
```

- Получаем сертификат: 
- Для Apache:
```bash
sudo certbot --apache
```

- Для Nginx (Нужно настроить виртуальный хост), но для nginx я пока что не делал сам, поэтому только предполагаю что так.:
```bash
sudo certbot --nginx
```

#### Настройка автоматического продления

- Проверить работу автоматического обновления:
```bash
sudo certbot renew --dry-run
```

---

### 6. Рекомендации по безопасности и эксплуатации

- Оставьте одну рабочую сессию при тестировании SSH-изменений
- Проверьте security group провайдера, если сервер в облаке
- Рассмотрите установку fail2ban для защиты от brute-force атак
- Сделайте резервную копию конфигураций:
```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```
- Настройте мониторинг срока действия SSL-сертификатов
- Регулярно обновляйте систему и установленные пакеты:
```bash
sudo dnf update
```
