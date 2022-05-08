---
layout: post
title: Several (Arch) linux tips: SSH-keys, letsencrypt, log4j
---

In this post I describe several (Arch) linux tips, I used myself on my raspberry PI running Arch linux. Maybe they are also helpful for you.


## Using SSH keys to login to Linux in stead of user name/password
In most cases it is preferrable, and easier to use keys to login your Linux box with SSH.
For a very good and understandable explanation of the keys see https://wiki.archlinux.org/title/SSH_keys.

To enable logging in via keys the following commands have to be executed on client (NOT on SSH-server):
```bash
# Generate key pair with comment  comment saying which user created the key on which machine and when
ssh-keygen -C "$(whoami)@$(uname -n)-$(date -I)" #accept default location for storage of keys
# You can leave the passphrase empty, but that is less secure, see link above
# Now copy public key to you server 
ssh-copy-id -p <portno> username@remote-server.org #replace of course username en server name, and <portno> by SSH port used on server, if other than 22

```

## Renew letsencrypt certificates
```bash
#check that there is no process listening to port 80
sudo netstat -tulpn|grep 80
#otherwise stop that proces
#ensure RPI is reachable on port 80 from the internet
#set port forwarding for IPv4 port 80 to RPI:80
sudo certbot renew
```

## Scan for Log4j vulnerabilities
```bash
curl -L https://github.com/fox-it/log4j-finder/raw/main/log4j-finder.py -o log4j-finder.py
sudo python3 log4j-finder.py
```
 
