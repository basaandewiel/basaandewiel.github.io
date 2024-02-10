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

The problem with Nextcloud seemed to be that Nextcloud was using https. So I installed an SSL certificate of Letsencrypt in the proxy container. But is was still not working. This was caused by the fact that the proxy was now terminating the https connection, but was itself sending traffic in https to the nextcloud container. 
And nextcloud had the setting of forcing https active. So the nextcloud container redirected all trafic to https. This resulted in the error in the browser of *too many redirections*. After turning the forcing of https to off in the nextcloud container everything was working.


# Instructions
The host (RPI) I was using was running Arch linux, so the 'lxd' command in substituted by 'incus', but further there are no noticable (for me) differences between lxd and incus. For installing incus see my other post on this subject.

```
# set port forwaring for port 80 and 443 on your router to the RPI host, and use the same port number for the forwaring as was received (so 80->RPI:80 and 443->RPI:443).
# instructions on how to do this are dependent on your router (for OpenWRT is under under Networking->firewall)
```

## Set up reverse proxy in linux container
```
# On the host RPI
# launch container named proxy
incus launch images:ubuntu/22.04 proxy

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
#Logout from the container.
logout
```

## Create the Nextcloud linux container
Download Nextcloudpi, special version of nextcloud for the raspberry pi, from `https://github.com/nextcloud/nextcloudpi/releases`. I used the LXD version form arm64 architecture.

```
wget https://github.com/nextcloud/nextcloudpi/releases/download/v1.53.1/NextcloudPi_LXD_arm64_v1.53.1.tar.gz
incus image import NextcloudPi_LXD_arm64_v1.53.1.tar.gz --alias "nextcloudpi"
incus launch "nextcloudpi" ncp
```

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
Restart the NGINX reverse proxy. By restarting the service, NGINX reads and applies the new site instructions just added to /etc/nginx/sites-enabled.

```
sudo systemctl reload nginx
#Exit the proxy container and return back to the host.
logout
```
