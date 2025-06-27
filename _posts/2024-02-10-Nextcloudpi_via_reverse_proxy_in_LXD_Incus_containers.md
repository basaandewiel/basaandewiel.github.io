---
layout: post
title: Nextcloudpi via reverse proxy, both in LXD/Incus Linux containers
---
Used sources
* Installing containers for reverse proxy and webserver
  * https://www.linode.com/docs/guides/beginners-guide-to-lxd-reverse-proxy/
* Installing Letsencrypt certificates
  * https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal


# Goal
Have Nexcloud working in a Linux containter on my Raspberry Pi, where Nextcloud is accessable from the interent.


# Introduction
I always wanted to try Nextcloud on my Raspberry Pi 5. But I did not want to destroy my RPI5 that is also used as my production system for home automation.
So i ended up using Linux containers, widely known as LXD. 

I am using Arch Linux on my RPI, and on Arch linux LXD is replaced by **Incus**.

For installing Incus, see one of my other posts, but in contrast to that post Incus is now used in bridge mode.

# Networking
I have been struggling with the networking part. I tried following:
* set port forwarding on my router (Openwrt) for port 80 and port 443 to my RPI
* on the RPI set port forwarding via the **nft** firewall (that was installed together with incus, and replaced **iptables**, for port 80 and 443 to the container running Nextcloud

This did not work, probably because the router is doing NAT (translates public IP addresses to private IP addresses and vice versa. So I tried doing NAT via nft, but could net get it to work.

Finally I found out that I had to use a **reverse proxy** to direct external traffic to the container running Nextcloud.
And the best part is that I found instruction to install the reverse proxy also in a Linux container.
And this setup did work with a simple website served by **nginx** or **apache2**, but not with my Nextcloud container.

This was the working setup with a simple website, using port 80 (so no https):
* my openwrt router forwards port 80 and 443 to my RPI
* this traffic was routed to the Linux container that was running the reverse proxy 
* the container running the reverse proxy sended the traffic to the other Linux container running a simple website

`router <--> RPI <--> proxy container <--> webserver container`

The problem with Nextcloud seemed to be that Nextcloud was using https. So I installed an SSL certificate of Letsencrypt in the proxy container.
```
incus exec ncp -- bash
# ncp-config ;select networking, select letsencrypt, fill in and change 'no' to 'yes'
```
But is was still not working. This was caused by the fact that the proxy was now terminating the https connection, but was itself sending traffic in https to the nextcloud container. 
And nextcloud had the setting of forcing https active. So the nextcloud container redirected all trafic to https. This resulted in the error in the browser of *too many redirections*. After turning the forcing of https to off in the nextcloud container everything was working.
To turn forcing https off: 
```
incus exec ncp -- bash
#ncp-config
  * select CONFIG
  * select nc-httpsonly
  * change 'yes' to 'no'
  * save
```
Now I can reach nextcloudpi via my own domain. However the browser indicats "not secure". However it indicates that the letsencrypt cerficate is valid.
**still have to find the solution for this**

# Instructions
The host (RPI) I was using was running Arch linux, so the 'lxd' command in substituted by 'incus', but further there are no noticable (for me) differences between lxd and incus. For installing incus see my other post on this subject.


## Set port forwarding in your router to the RPI
```
# set port forwaring for port 80 and 443 on your router to the RPI host, and use the same port number for the forwaring as was received (so 80->RPI:80 and 443->RPI:443).
# instructions on how to do this are dependent on your router (for OpenWRT is under under Networking->firewall)
```

## Set up reverse proxy in linux container
```
# On the host RPI
# launch container named proxy
# old version - incus launch images:ubuntu/22.04 proxy
incus launch images:ubuntu/plucky/arm64 proxy


# Add LXD proxy devices to redirect connections from the internet to ports 80 (HTTP) and 443 (HTTPS) on the server to the respective ports at the proxy container.
incus  config device add proxy myport80 proxy listen=tcp:0.0.0.0:80 connect=tcp:127.0.0.1:80 proxy_protocol=true
incus config device add proxy myport443 proxy listen=tcp:0.0.0.0:443 connect=tcp:127.0.0.1:443 proxy_protocol=true
# note the IPv4 and IPv6 address of the proxy, you need them later; they are shown via
incus list

#Start a shell in the proxy container.
incus exec proxy -- bash
#Update the package list.
apt update
#Install NGINX in the container.
apt install -y nginx

# set max body size; this is default 1M; otherwise file uploads to file servers behind the proxy (so also for nextcloud) are not possible
vim /etc/nginx/nginx.conf
# add following line in server block (without "#")
# client_max_body_size 8m;

# restart nginx
systemctl reload nginx


#Logout from the container.
logout
```

## Create the Nextcloud linux container
Download Nextcloudpi, special version of nextcloud for the raspberry pi, from `https://github.com/nextcloud/nextcloudpi/releases`. I used the LXD version for arm64 architecture.

```
wget https://github.com/nextcloud/nextcloudpi/releases/download/v1.53.1/NextcloudPi_LXD_arm64_v1.53.1.tar.gz
# import image in incus, and call the image nextcloudpi
incus image import NextcloudPi_LXD_arm64_v1.53.1.tar.gz --alias "nextcloudpi"
# launch the container and call it ncp
incus launch "nextcloudpi" ncp
```

## Configure nextcloudpi
To be able to access nextcloudpi, add the domain you use for nextcloudpi, to the list of trusted domains.
I think this was done via the `ncp-config` command. This command edit the contents of `/var/wwww/nextcloud/config/config.php`.
When you manually edit this config, it will be overwritten at boot up of the ncp container.
In a working config the trusted domain and trusted proxies have following contents

```
  'trusted_domains' =>
  array (
    0 => 'localhost',
    7 => 'nextcloudpi',
    5 => 'nextcloudpi.local',
    8 => 'nextcloudpi.lan',
    3 => 'ncp.YOURDOMAIN.EXT',
    11 => 'YOURPUBLIC IP ADDR',
    1 => 'IP ADDR OF NCP CONTAINER',
    14 => 'NAME OF NCP CONTAINER',
    15 => 'ncp.YOURDOMAIN.EXT',
  ),
  'trusted_proxies' =>
  array (
    1 => 'IP ADDR OF PROXY CONTAINER',
    11 => '127.0.0.1',
    12 => '::1',
    0 => 'NAME OF PROXY CONTAINER',
```
In the same file also add the name of the proxy container 'proxy' to the list of trusted_proxies


## Direct Traffic to the Nextcloud container from the Reverse Proxy
Start a shell in the proxy container.
```
incus exec proxy -- bash
```
Create the file nextcloud.yourdomain.com in /etc/nginx/sites-available/ for the configuration of your first website.
NB: replace nextcloud.yourdomain.com with the external url via wich nextcloud should be made available. I use a free domain from afraid.org, and created a subsub domain for my nextcloud.


File: nextcloud.yourdomain.com
```
server {
        listen 80 proxy_protocol;
        listen [::]:80 proxy_protocol;

        server_name nextcloud.yourdomain.com;

        location / {
                include /etc/nginx/proxy_params;

                proxy_pass http://proxy;
        }

        real_ip_header proxy_protocol;
        set_real_ip_from 127.0.0.1;
}
```

Enable the website.
```
sudo ln -s /etc/nginx/sites-available/nextcloud.yourdomain.com /etc/nginx/sites-enabled/
`#Restart the NGINX reverse proxy. By restarting the service, NGINX reads and applies the new site instructions just added to /etc/nginx/sites-enabled.
sudo systemctl reload nginx
`#Exit the proxy container and return back to the host.
logout
```

### Configure backup of your Nextcloud data
* first create a directory that is accessible from both the backupserver (in my case my RPI5 which also runs all the incus containers) and from with the ncp linux container
  * on RPI5
```
incus config device add ncp ncpBackup disk source=/mnt/drivea/backups/ncp path=/ncpBACKUP shift=true
incus exec ncp -- ls /ncpBACKUP
`# /ncpBackup: directory where this shared directory is available from within ncp linux container
`# /mnt/drivea/backups/ncp: directory where the shared directory is available from within the RPI5
```

nextcloudPI has several 'own' scripts in `/usr/local/bin`
With `/usr/local/bin/ncp-backup` you can back up all contents of ncp to a file and optionally include data dir and compress it.
I have tested this, by creating a backup from ncp, and importing this backup in fresh created other incus containter called 'ncprestored'

On RPI5
`incus exec proxy -- bash`

In ncp container:
`ncp-config->backup->include data and compress`

On RPI5:
`incus exec proxy -- bash`

In proxy container, let proxy point to incus container in which backup will be restored:
Change contents of  `/etc/nginx/sites-enabled/ncp.aandewiel.eu`
replace `ncp` with `ncprestored` (name of incus container to which the backup will be restored)
`systemctl reload nginx`

On RPI5:
```
incus stop ncp
incus start ncprestored

incus exec ncprestored -- bash
```

In ncprestored:
`ncp-config->backup->restore`
* select /ncpBackup/<file>
After this restore operation I only had to do one change:
run `ncp-config` and 
set `forced-https` to `off`.

All ncp users and all data was restored.


### Create cron job for regular automatic backups
* run ncp-config and select 'backupsday' to '1' to create a backup every day
This command creates file '/etc/cron.d/ncp-backup-auto' with following contents:
`0  3  */7  *  *  root  /usr/local/bin/ncp-backup-auto`

It is also good practice and handy to let sync 'your' files in ncp to for instance your (Windows) laptop. This
can be done by installing app `Nextcloud` on Windows (only tested on windows).
After this sync you can also edit, move, add and delete ncp file pertaining to your account.



## Debugging
For debugging you have several options, some are:
* check ports the system listens to `ss -tlpn`; 
* to check whether the traffic arrives at the RPI from the router you can use for instance 'tcpdump'
* check the status of services, for instance of nginx via `systemctl status nginx` 

