## О сертификатах Let`s Encrypt

Пост считается полным, но однозначно будет претерпевать изменения для более удобного восприятия материала. 
Let's Encrypt — это некоммерческий центр сертификации, который предоставляет **бесплатные SSL/TLS-сертификаты**. Эти сертификаты имеют срок действия **90 дней**, но процесс обновления можно автоматизировать.

Сертификаты Let's Encrypt признаются всеми основными браузерами и обеспечивают такое же шифрование, как и платные аналоги. Основное отличие — это короткий срок действия и необходимость автоматического обновления.

### Мои шаги при получении сертификата Let`s Encrypt

#### sudo certbot --apache

Сначала я ввел команду `sudo certbot --apache`.
> В надежде что всё будет настроено автоматически, почему-то я был уверен что 
certbot настроит сам и виртуальный хост для Apache. Но в случае с AlmaLinux 8, я ошибся про автоматическую настройку
виртуального хоста, в дальнейшем мне пришлось настраивать виртуальный хост самому, хотя это и не сложно.

В Almalinux 8 на этом шаге я получаю ошибку: 
```bash
[user@host ~]$ sudo certbot --apache
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Error while running apachectl configtest.

AH00526: Syntax error on line 85 of /etc/httpd/conf.d/ssl.conf:
SSLCertificateFile: file '/etc/pki/tls/certs/localhost.crt' does not exist or is empty

Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel):
```

#### Анализ "*Error while running apachectl configtest*"

- Certbot перед работой запускает `apachectl configtest` для проверки конфигурации

- Apache находит ошибку в файле /etc/httpd/conf.d/ssl.conf

- На строке 85 указан сертификат, который отсутствует или пустой

---


Но после этой ошибки, как видно cerbot предлагает ввести e-mail. Соглашаемся в надежде что всё же он в конце всё сам правильно настроит, а ошибку 
*Error while running apachectl configtes* пропустим, так как в выводе видно, что связана она с тем что в конфигурации прописан файл которого нет.
Нам это сейчас не важно, нам важно получить сертификат.

#### Нет виртуального хоста на 80 порту

Вводим e-mail, соглашаемся там со всякими условиями и прочее, доходим до шага, где вводим домены для которых получаем сертификаты, 
и получаем что-то вроде ошибки, о том что cerbot не находит виртуальный хост на 80 порту, который ему нужен, чтобы подтвердить сертификат.
```bash
space separated) (Enter 'c' to cancel): linuxgik.ru www.linuxgik.ru
Requesting a certificate for linuxgik.ru and www.linuxgik.ru
Unable to find a virtual host listening on port 80 which is currently needed for Certbot to prove to the CA that you control your domain. Please add a virtual host for port 80.
Ask for help or search for solutions at https://community.letsencrypt.org. See the logfile /var/log/letsencrypt/letsencrypt.log or re-run Certbot with -v for more details.
```

##### Создаем виртуальный хост
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

##### Получаем SSL-сертификат
```bash
sudo certbot --apache -d linuxgik.ru -d www.linuxgik.ru
```
Проверяем результат:
```bash
# Проверяем HTTPS
curl -I https://linuxgik.ru

# Проверяем редирект HTTP → HTTPS
curl -I http://linuxgik.ru
```

