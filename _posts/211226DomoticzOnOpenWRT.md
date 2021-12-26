---
layout: post
title: Install Domoticz on router running OpenWRT
---
# Domoticz on OpenWRT
I was running domoticz on my Raspberry PI (RPI), but because I also did some development work on the RPI, sometimes the RPI crashed. And my Domoticz should always be running, because of all home automation running on it.

I also had a Linksys 3200 ACM router, running OpenWRT, with enought empty space. So I wanted to install domoticz on it, including a Zwave USB-stick.
The code below installs everything on OpenWRT that is neede to run domoticz. 

NB: I store the domoticz database on my RPI to prevent space problems on my router.


```bash
#This script can be used to install domoticz on openWRT
#tested on 211203 with OpenWRT 20.01
#can also be used to copy needed scripts/files vanished after reboot
#NB: domoticz.db is stored via softlink to mounten RPI3/motion share on RPI3
#Folowing must be CHANGED
# * IP-number of the device where the database is stored
# * username and password for mounting the share
IPADDRESS = 192.168.1.13
USER=username
PASSWORD=password

#install required packages
opkg update
opkg install domoticz
#install for USB ACM port for ZWave stick
opkg install kmod-usb-acm
#openzwave is only needed if you have a zwave USB-stick for instance for a sirene
opkg install openzwave
chmod 777 /dev/ttyACM0

#install curl; needed to send telegram message in case of alarm (implemented in lua scripts)
opkg install curl #needed for sending telegram message ico burglar

#The domoticz.db is not saved on openWRT, because it grows constantly. To limit the size I set the history for sensors to 10 days. But I want a long history from the smart meter readings, so the database will grow.
#To prevent storage problems I store domoticz.db on my Raspbery PI3 with attached hard disk.
#mount share on RPI, and put domoticz database on that share
opkg install kmod-fs-cifs kmod-nls-base
mkdir /mnt/share
#
mount -t cifs //$IPADRESS/motion /mnt/share/ -o username=$USER,password=$PASSWORD
cd /var/lib/domoticz
ln -s /mnt/share/domoticz.db domoticz.db #create soft link to db on rpi3

#restore /etc if necessary
cd /etc/config/
#create file /etc/config/domoticz @@@
#create file /etc/init.d/domoticz @@@

#restore /var/lib/domoticz if necessary -seems to survive reboots, but probably not when device is turned off and on

#start domoticz
/etc/init.d/domoticz start
```
