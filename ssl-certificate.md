В данной инструкции объясняется, как создать SSL-сертификат для домена (доменов) с помощью [Certbot](https://certbot.eff.org/).

Для создания самоподписанного (self-signed) сертификата, используйте [этот](https://github.com/lstellway/self-signed-ssl) скрипт или [acert](https://github.com/lstellway/acert). Созданный сертификат нужно добавить в список доверенных сертификатов в хроме:

1. Откройте раздел сертификатов в настройках: `chrome://settings/certificates` (`Chrome settings > Security > Manage certificates`).
2. Откройте вкладку "Authorities".
3. Нажмите "Import" и укажите созданный файл certificate authority.

## Установка

Выжимка из [документации](https://certbot.eff.org/instructionshttps://certbot.eff.org/docs/using.html) (Ubuntu 20.04 + nginx):

```bash
$ sudo snap install --classic certbot
$ sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

## Создание сертификата

```bash
$ certbot --nginx -d example.com
```

## Сертификат для всех поддоменов (wildcard)

Чтобы не получать сертификаты отдельно для каждого поддомена, можно настроить сертификат для всех поддоменов. Certbot позволяет сделать это, если использовать получение сертификата через DNS.

Прежде всего необходимо установить плагин для вашего провайдера DNS. Например, для DigitalOcean это [certbot-dns-digitalocean](https://certbot-dns-digitalocean.readthedocs.io/en/stable/):

```bash
$ sudo snap install certbot-dns-digitalocean
```

Создайте файл `certbot-digitalocean.ini` с токеном API DigitalOcean и установите на него права с помощью следующей команды:

```bash
$ chmod 600
```

Затем создайте файл `create-certificate.sh`:

```bash
#!/bin/bash

certbot certonly \
  --dns-digitalocean \
  --dns-digitalocean-credentials ./certbot-digitalocean.ini \
  -d domain.com \
  -d *.domain.com
```

Осталось дать права на выполнение файла и запустить его:

```bash
$ chmod +x create-certificate.sh
$ ./create-certificate.sh
```

### Ссылки

- [https://certbot.eff.org/lets-encrypt/ubuntufocal-nginx](https://certbot.eff.org/lets-encrypt/ubuntufocal-nginx)
- [https://certbot.eff.org/docs/using.html#dns-plugins](https://certbot.eff.org/docs/using.html#dns-plugins)
- [https://certbot-dns-digitalocean.readthedocs.io/en/stable/](https://certbot-dns-digitalocean.readthedocs.io/en/stable/)
- [https://www.digitalocean.com/community/tutorials/how-to-create-let-s-encrypt-wildcard-certificates-with-certbot](https://www.digitalocean.com/community/tutorials/how-to-create-let-s-encrypt-wildcard-certificates-with-certbot)

## Обновление сертификата

Для автоматического обновления сертификата необходимо настроить запуск следующего скрипт каждый месяц с помощью Cron или Jobber:

```bash
#!/bin/bash

sudo certbot renew --nginx
```

В случае, если настроили получение сертификата через DNS: 

```bash
#!/bin/bash

certbot renew \
  --dns-digitalocean \
  --dns-digitalocean-credentials ./certbot-digitalocean.ini
```

Не забудьте дать права на выполнение файла:

```bash
$ chmod +x renew-certificate.sh
```
