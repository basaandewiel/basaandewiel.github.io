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

When you are asked for a database name enter: kimaidb

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

## Backup your kimai database
Of course you want to regulary backup your kimai database; you can do this as follows:
```bash
mysqldump --single-transaction -u kimai -p -h 127.0.0.1 kimaidb > ~/kimaidb-`date +%F_%H-%M`.sql
```
You must enter the corresponding password.

Of course you want the backups to be made via a script that is called by the cron.
Create following script:
```bash
#!/bin/bash

export CONNECTION_CONFIG=/home/baswi/.kimai2.cnf
export KIMAI_DIR=/srv/http/kimai2
export BACKUP_DIR=/home/baswi/backups/kimai

export DATE=`date +%F_%H-%M`
export BACKUP_STORAGE_DIR=$BACKUP_DIR/storage
export BACKUP_TMP_DIR=$BACKUP_DIR/temp/$DATE

mkdir -p $BACKUP_STORAGE_DIR
mkdir -p $BACKUP_TMP_DIR
mkdir -p $BACKUP_TMP_DIR/var/data/
mkdir -p $BACKUP_TMP_DIR/var/plugins/

mysqldump --defaults-file=$CONNECTION_CONFIG --single-transaction --no-tablespaces -u kimai -h 127.0.0.1 kimaidb > $BACKUP_TMP_DIR/kimai2-$DATE.sql

cp $KIMAI_DIR/.env $BACKUP_TMP_DIR/
cp -R $KIMAI_DIR/var/data/* $BACKUP_TMP_DIR/var/data/
cp -R $KIMAI_DIR/var/plugins/* $BACKUP_TMP_DIR/var/plugins/

if [[ -d "$KIMAI_DIR/var/invoices/" ]];
then
          mkdir -p $BACKUP_TMP_DIR/var/invoices/
           cp -R $KIMAI_DIR/var/invoices/* $BACKUP_TMP_DIR/var/invoices/
fi

if [[ -d "$KIMAI_DIR/var/export/" ]];
then
          mkdir -p $BACKUP_TMP_DIR/var/export/
           cp -R $KIMAI_DIR/var/export/* $BACKUP_TMP_DIR/var/export/
fi

pushd $BACKUP_TMP_DIR
tar -czf $BACKUP_STORAGE_DIR/$DATE.tgz * .env
//clean up
cd ..
rm -rf $DATE //remove the temp subdir
//delete backups older than 60 days
find $BACKUP_STORAGE_DIR/* -atime +60 -type f -delete
popd
```

Add the script to the cron, for instance:

`52 8,18 * * *   /home/user/scripts/backup_kimai.bash`
To backup the Kima database and other important files, two times a day.