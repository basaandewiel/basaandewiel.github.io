---
layout: post
title: Yocto on Arch linux 
---
## Situation
I wanted to know more from Yocto, so I installed in on Arch Linux first.

```bash
# install required packages
sudo pacman -S rpc@@@

git clone https://git.yoctoproject.org/git/poky
cd poky
git clone https://github.com/aehs29/meta-freertos.git
source oe-init-build-env
bitbake freertos-demo #takes about 45 minutes on old laptop
#
# add networking, virtual interface etc
# thanks to https://bbs.archlinux.org/viewtopic.php?id=207907
sudo ip link add name br0 type bridge
sudo ip addr add 172.20.0.1/16 dev br0
sudo ip link set br0 up
sudo dnsmasq --interface=br0 --bind-interfaces --dhcp-range=172.20.0.2,172.20.255.254
# reboot now, I you have an updated kernel and not yet rebooted
sudo modprobe tun
sudo [[ ! -d /etc/qemu ]] && mkdir /etc/qemu
sudo echo allow br0 > /etc/qemu/bridge.conf
sudo sysctl net.ipv4.ip_forward=1
sudo sysctl net.ipv6.conf.default.forwarding=1
sudo sysctl net.ipv6.conf.all.forwarding=1
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i tap0 -o wlan0 -j ACCEPT
#
sudo runqemu-gen-tapdevs 1000 1000 4 build/tmp/sysroots-components/x86_64/qemu-helper-native/usr/bin/
#
cd build
runqemu nographic
#to quit qemu enter: ctrl-a followed by 'c', and enter 'quit'
```

edit local/conf.local and add/change
```bash
MACHINE ?= "qemux86-64"
ONNECTIVITY_CHECK_URIS = "https://www.google.com/"

```
