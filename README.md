# Создание сервера и первоначальная настройка

ОС: Ubuntu 20.04

1. Расскомментируйте строку `force_color_prompt=yes` в `.bashrc`, чтобы включить цвета для prompt в терминале.

Добавьте следующую строку в конец `.bashrc`, чтобы в приветсвии терминала отобраажалась текущая ветка Git:

```bash
PS1='\[\e]0;\u@\h: \w\a\]${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\] $(__git_ps1 "(%s)")\$ '
```

Выполните команду `source .bashrc`, чтобы обновить конфиг.

Обновите пакеты:

```bash
apt update
apt upgrade
apt dist-upgrade
apt autoremove
```

Изменить порт SSH в файле `/etc/ssh/sshd_config` (значение Port) и перезапустите сервис SSH.

Изменить пароль для пользователя root с помощью команды `passwd`.

## Создание swap-раздел

Добавить в конец файла `/etc/sysctl.conf`:

```bash
vm.swappines=10
vm.vfs_cache_pressure=50
```

Перезагрузить сервер с помощью команды `reboot`.

### Ссылки

[https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-16-04)

## Создание нового пользователя

Необходимо создать нового пользователя, чтобы использовать его вместо пользователя root.

```bash
useradd -m -s/bin/bash muhammad
```

- `-m` - создать домашнюю директорию для пользователя
- `-s` - задает shell

Изменить пароль для созданного пользователя:

```bash
passwd muhammad
```

Сделать пользователя sudoer:

```bash
usermod -aG sudo muhammad
```

Переключить на созданного пользователя:

```bash
sudo su muhammad
```

Изменить привествие терминала в `.bashrc` (`PS1`), как описано выше.

[Сгенерировать публичный ключ](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04) для созданного пользователя:

```bash
ssh-keygen
```

## Установка и настройка ПО

Установить:

- unzip
- cURL
- Git
- imagemagick
- PHP с расширениями bcmath, curl, gd, gmp, json, imagick, intl, mysql, pgsql, sqlite, xml, zip, php-imagick, php-redis
    - `sudo apt install software-properties-common`
    - `sudo add-apt-repository ppa:ondrej/php`
- nginx
- PostgreSQL (если не используется managed database)
- build-essential (команда make)
- Redis (key-value хранилище для сессий, кеша и множества других вещей)
- net-tools (для команды netstat)
- Node.js (с помощью [nodesource.com](http://nodesource.com/))
    - [https://github.com/nodesource/distributions#installation-instructions](https://github.com/nodesource/distributions#installation-instructions)
- Composer

Удалить Apache

```bash
apt purge apache2
```

## Настройка nginx и PHP

- Увелить размер загружаемых файлов в nginx (`client_max_body_size`) и PHP (`post_max_size`, `upload_max_filesize`) до 10 Мб
- Включить gzip в nginx.conf ([https://www.digitalocean.com/community/tutorials/how-to-add-the-gzip-module-to-nginx-on-ubuntu-14-04](https://www.digitalocean.com/community/tutorials/how-to-add-the-gzip-module-to-nginx-on-ubuntu-14-04))
- Увеличить память, доступную PHP (`memory_limit`) до 256 Мб
- Отключить выдачу системной информации в заголовках ответов веб-сервера в файле `/etc/nginx/nginx.conf`:

```bash
http {
    ...
    server_tokens off;
    ...
}
```

Добавить конфиг nginx для сайта.

[Настроить SSL-сертификаты](ssl-certificate.md) и их обновление.

При необходимости настроить базовую авторизацию:

```bash
sudo apt install apache2-utils
sudo mkdir /etc/nginx/htpasswd
sudo htpasswd -c /etc/nginx/htpasswd/mysite <username>
```

- [https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-http-basic-authentication](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-http-basic-authentication/)
- [https://www.digitalocean.com/community/tutorials/how-to-set-up-password-authentication-with-nginx-on-ubuntu-14-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-password-authentication-with-nginx-on-ubuntu-14-04)

Настроить перезагрузку сервиса PHP без sudo (для Deployer). Для этого нужно созать файл `<username>-restart-php` (например, `johndoe-restart-php`) в папке `/etc/sudoers.d`:

```php
muhammad ALL=(ALL) NOPASSWD:/usr/sbin/service php8.0-fpm restart
```

Не забудьте заменить версию PHP, если версия отличается.

## Jobber

Установить и настроить [Jobber](https://dshearer.github.io/jobber/).

```yaml
version: 1.4

prefs:
    logPath: /home/muhammad/jobber.log

jobs:
    daily_database_backup:
        cmd: php /home/muhammad/database-backup-manager/backup.php > /home/user/database-backup-manager/daily-backups.log
        time: 0 30 0 * * *

    renew_ssl_certificates:
        cmd: bash /home/muhammad/certbot/renew-certificate.sh
        time: 0 0 10 * * *

    run_scheduler:
        cmd: php /home/muhammad/websites/my-site/current/artisan schedule:run >> /dev/null 2>&1
        time: 0 */1 * * * *
```

```bash
chmod 755 .jobber
```

## База данных

Лучше использовать какой-нибудь DBaaS (Database-as-a-Service), типа [Amazon RDS](https://aws.amazon.com/ru/rds/) или [Digital Ocean Managed Databases](https://www.digitalocean.com/products/managed-databases/), но если все же возникла необходимость настроить что-то свое, можно воспользоваться инструкциями ниже.

Изменить пароль для пользователя postgres

```bash
sudo -u postgres psql
```

```sql
ALTER USER postgres PASSWORD 'new_strong_password';
```

Создать отдельного пользователя PostgreSQL.

```sql
CREATE USER muhammad WITH PASSWORD '123456';
```

Создать БД:

```sql
CREATE DATABASE mydb OWNER muhammad;
```

Создать отдельную схему, чтобы использовать ее вместо public:

```sql
CREATE SCHEMA foobar;
```

[https://wiki.postgresql.org/wiki/A_Guide_to_CVE-2018-1058:_Protect_Your_Search_Path](https://wiki.postgresql.org/wiki/A_Guide_to_CVE-2018-1058:_Protect_Your_Search_Path)

Настроить доступа в файле pg_hba.conf.
Настроить фаерволл с помощью UFW. Разрешить доступ к базе для прода и к SSH с любого IP. Если порт SSH был изменен, нужно учитывать это. Если порт SSH был изменен, нужно открыть его (tcp).

```bash
sudo ufw allow from any to any port 5600 proto tcp comment "SSH"
sudo ufw allow from 10.120.0.2 to any port 5432 proto tcp comment "PostgreSQL"
sudo ufw allow http/tcp
sudo ufw allow https/tcp

sudo ufw status verbose
sudo ufw status numbered

sudo ufw delete 1
```

[https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-20-04-ru](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-20-04-ru)[https://linuxize.com/post/how-to-setup-a-firewall-with-ufw-on-ubuntu-18-04](https://linuxize.com/post/how-to-setup-a-firewall-with-ufw-on-ubuntu-18-04)[https://www.8host.com/blog/zashhita-postgresql-ot-avtomatizirovannyx-xakerskix-atak/](https://www.8host.com/blog/zashhita-postgresql-ot-avtomatizirovannyx-xakerskix-atak/)

## Прочее

- Установить и настроить Deployer.
    - `sudo apt-get install acl`
    - [https://github.com/deployphp/deployer/issues/1118](https://github.com/deployphp/deployer/issues/1118)
- Настроить Supervisor для запуска очереди Laravel ([https://laravel.com/docs/7.x/queues#supervisor-configuration](https://laravel.com/docs/7.x/queues#supervisor-configuration)).

## Опционально

- Установить [adminer](https://www.adminer.org/)
- Установить PgBouncer
- Настроить бэкапы
- Настроить UpTime Monitor
