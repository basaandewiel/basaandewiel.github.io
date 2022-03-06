---
layout: post
title: Install Domoticz on router running OpenWRT
---
## Used configuration
- hardware: Linksys WRT3200ACM
- firmware version: OpenWrt 21.02.1 r16325-88151b8303 / LuCI openwrt-21.02 branch git-21.357.58218-b3cd473


I was running domoticz on my Raspberry PI (RPI), but because I also did some development work on the RPI, sometimes the RPI crashed. And my Domoticz should always be running, because of all home automation running on it.

I also had a Linksys 3200 ACM router, running OpenWRT, with enought empty space. So I wanted to install domoticz on it, including a Zwave USB-stick.
The code below installs everything on OpenWRT that is neede to run domoticz. 

NB: I store the domoticz database on my RPI to prevent space problems on my router.
This is also why I turned off all logging of Domoticz (in /etc/config/domoticz - see below)

The easiest way to install domoticz is by executing commands via ssh. For this you need to ssh into your openWRT router (google to find out how to do that).

```bash
#This script can be used to install domoticz on openWRT
#tested on 211203 with OpenWRT 20.01
#NB: domoticz.db is stored via softlink to mounted share on RPI/SMBshare 
#Folowing must be CHANGED in this script:
# * IP-number of the device where the database is stored
# * username and password for mounting the share
IPADDRESS = 192.168.1.13
USER=username
PASSWORD=password
SMBshare=<name of SMB share on RPI>

#install required packages
NB: it seems that sometimes installing via CLI gives message that not enough sspace is left, and installing vai Luci does succeed.

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
mount -t cifs //$IPADRESS/SMBshare /mnt/share/ -o nolock,username=$USER,password=$PASSWORD
cd /var/lib/domoticz
ln -s /mnt/share/domoticz.db domoticz.db #create soft link to db on RPI
ln -s /mnt/share/domoticz_backups backups #create soft link to backup directory used for hourly,daily,monthly backups by domoticz
# of course /mnt/share/domoticz_backups mus be created first on RPI

#NB: the option 'nolock' in the mount command is necessary, otherwise domoticz will crash every hour (see also below)
```

Now you need to create a config file for domoticz. Put this file in `/etc/config/domoticz`

```bash
config domoticz
    option disabled '0'
    option loglevel '0'
    option syslog 'daemon'
    # option sslcert '/path/to/ssl.crt'
    # option sslkey '/path/to/ssl.key'
    # option sslpass 'passphrase'
    # option ssldhparam '/path/to/dhparam.pem'
    option sslwww '0'
    # CAUTION - by default, /var is not persistent accross reboots
    # Don't forget the trailing / - domoticz requires it
    option userdata '/var/lib/domoticz/'
#    option userdata '/home/domoticz/'
#config device
#       option product '658/200/0'
#       option symlink 'ttyACM-aeotec-zstick-g5'
#config device
#       option serial '526359'
#       option symlink 'ttyUSB-serial'
#config device
#       option usbif '2-1:1.0'
#       option symlink 'ttyUSB-port1'
#config device
#       option product '67b/2303/202'
#       option usbif '2-2:1.0'
#       option symlink 'ttyUSB-port2'

```

Noy need to create one other file, file for starting domoticz as a service. Put this file in `/etc/init.d/domoticz`

```bash
#!/bin/sh /etc/rc.common
START=99
USE_PROCD=1
#baswi inserted next line
PROCD_DEBUG=1
PROG=/usr/bin/domoticz
PIDFILE=/var/run/domoticz.pid
start_domoticz() {
    local section="$1"
    local loglevel www log sslcert sslpass sslwww syslog userdata
    config_get loglevel "$section" "loglevel"
    config_get sslcert "$section" "sslcert"
    config_get sslkey "$section" "sslkey"
    config_get sslpass "$section" "sslpass"
    config_get ssldhparam "$section" "ssldhparam"
    config_get sslwww "$section" "sslwww"
    config_get www "$section" "www"
    config_get log "$section" "log"
    config_get syslog "$section" "syslog"
    config_get userdata "$section" "userdata"
    [ -n "$loglevel" ] && procd_append_param command -debuglevel "$loglevel"
    [ -n "$syslog" ] && procd_append_param command -syslog "$syslog"
    [ -n "log" ] && procd_append_param command -log "$log"
    [ -d "${userdata}" ] || {
        mkdir -p "${userdata}"
        findLatestDatabase "$userdata"
        chmod 0770 "$userdata"
        chown root:root "$userdata"
    }
    # By default, ${userdata}/scripts is a symlink to /etc/domoticz/scripts
    # and the two dzVents directories under there which Domoticz will actually
    # write to at runtime are symlinked back to /var/lib again.
    [ -d "${userdata}/plugins" ] || ln -sf /etc/domoticz/plugins "${userdata}/plugins"
    [ -d "${userdata}/scripts" ] || ln -sf /etc/domoticz/scripts "${userdata}/scripts"
    for DIR in data generated_scripts; do
        [ -d /data/domoticz/dzVents/$DIR ] || {
            mkdir -p /data/domoticz/dzVents/$DIR
            chown root.root /data/domoticz/dzVents/$DIR
        }
    done
    procd_append_param command -userdata "$userdata"
    procd_append_param command -www "$www"
    [ -n "$sslcert" -a "${sslwww:-0}" -gt 0 ] && {
        procd_append_param command -sslcert "$sslcert"
            [ -n "$sslkey" ] && procd_append_param command -sslkey "$sslkey"
        [ -n "$sslpass" ] && procd_append_param command -sslpass "$sslpass"
        [ -n "$ssldhparam" ] && procd_append_param command -ssldhparam "$ssldhparam"
    } || procd_append_param command -sslwww "$sslwww"
}
start_service() {
    procd_open_instance
    procd_set_param command "$PROG"
#baswi - added next line; https://openwrt.org/docs/guide-developer/procd-init-scripts
#should be necessary for service trigger
    procd_set_param netdev
    procd_append_param command -noupdates
    procd_append_param command -approot /usr/share/domoticz/
#       procd_append_param command -wwwbind 192.168.1.1
    config_load "domoticz"
    config_get_bool disabled "$section" "disabled" 0
    [ "$disabled" -gt 0 ] && return 1
    config_foreach start_domoticz domoticz
    procd_set_param pidfile "$PIDFILE"
    procd_set_param respawn
    procd_set_param stdout 0
    procd_set_param term_timeout 10
    procd_set_param user "root"
    procd_close_instance
}
findLatestDatabase() {
    destinationFolder=$1
    yourfilenames=`ls /root/domoticz*[0-9].db`
    LatestDate=0
    for eachfile in $yourfilenames
    do
        echo $eachfile
        if [[ "${eachfile//[!0-9]/}" > "$LatestDate" ]]; then
            LatestDate=${eachfile//[!0-9]/}
        fi
    done
    if [ "$LatestDate" -gt "0" ]; then
        cp "/root/domoticz${LatestDate}.db" "${destinationFolder}domoticz.db"
    fi
}

#211222 added by baswi
service_triggers() {
#reload after 60 seoncds; default was restarting every 15 seconds for max 6 times
#  PROCD_RELOAD_DELAY=60000
# retart domoticz if RPI3 (on wich domoticz.db resides) is back
   procd_add_interface_trigger "interface.*.up" "lan1" /etc/init.d/domoticz restart
}

```

Now you can start Domoticz:
```bash

#start domoticz
/etc/init.d/domoticz start
```

Check whether domoticz starts by going to <ip-of-router:8080>. Domoticz should load, and when you have correctly linked the domoticz.db file to a existing database file, you should see all the devices in your domoticz.

NB: with this configuration domoticz is installed in RAM disk; my experience is that is does survice reboots of the router. You have probably to reinstall domoticz after power failure, of after power cycling the router (NOT yet tested).

## Using Lua scripts
If you use domoticz scripts (lua for instance) then you have to copy your scripts to `/etc/domoticz/scripts/` .
If you want to use persistant data in these scripts, then you have to create following directory: 
`mkdir /var/lib/domoticz/dzVents; mkdir /var/lib/domoticz/dzVents/data`
This directory is used to store persistant data; 
NB: this persistant data is in RAM disk, but is it persistant across subsequent call to the dzVents script.


## Domoticz crashes when autobackupping to a network drive 
Source used: https://github.com/domoticz/domoticz/issues/4180

When the domoticz.db is on a CIFS mounted drive, as here is the case, the drive must be mounted with the "-o nolock" option. Otherwise Domoticz cannot lock the database when it wants to make a backup of the database, and then Domoticz crashes after 5 minutes of retrying.

## Known issues/things not working
- mailing from domoticz (probably you have to install mail client on OpenWRT)
After a few days I saw that domoticz crashes every hour at xx.05. The was caused by turning on the auto backup feature in Domoticz, and letting it make a backup to a network drive (mounted as CIFS, see above).
