---
layout: post
title: Raspberry PI (3) - Arch linux
---
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
 
