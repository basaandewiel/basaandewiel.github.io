---
layout: post
title: Install kimai (hour registration) on RPI running Arch linux
---
## install kimai
```bash
cd /srv/http/
git clone -b 1.16.9 --depth 1 https://github.com/kevinpapst/kimai2.git
cd kimai2/
composer install --no-dev --optimize-autoloader
```

Edit .env file:
`vim .env`
change DBURL to: DATABASE_URL=mysql://kimai:kimai@127.0.0.1:3306/kimaidb?charset=utf8&serverVersion=mariadb-10.5.8

Now execute follwoing commands:

```bash
bin/console kimai:install -n
sudo systemctl restart nginx
cd /etc/nginx/conf.d/
sudo vim kimai.conf
sudo systemctl restart nginx
cd /srv/http/
sudo chown -R http:http kimai2/
cd kimai2/
sudo bin/console kimai:user:password "$USER, $PASSW"
sudo bin/console kimai:user:create $USER $EMAIL ROLE_SUPER_ADMIN
cd var/log/
cd var
sudo chmod o+w sessions/
```
