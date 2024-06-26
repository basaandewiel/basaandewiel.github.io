---
layout: post
title: Yocto on Arch linux (running first build on qemu)
---
With Yocto you can create your own embedded linux distribution. So **you** can decide what will be incorporated in the linux image and what not. You can customise your linux image and optimise it for the embedded system you are using.

To try Yocto I build my own linux distro and run it via qemu.

## Instructions
Install Yocto on Arch linux and build your first target and run it via qemu

```bash
# install required packages
sudo pacman -S chrpath diffstat rpcsvc-proto

git clone https://git.yoctoproject.org/git/poky
cd poky
git clone https://github.com/aehs29/meta-freertos.git
source oe-init-build-env
bitbake-layers add-layer ../meta-freertos
```
Now you are in directory `~/poky/build/
Change conf/conf.local
```bash
echo "MACHINE = \"qemuarmv5\"" >> ./conf/local.conf
echo "DISTRO = \"freertos\"" >> ./conf/local.conf
echo "CONNECTIVITY_CHECK_URIS = \"https://www.google.com\/" >> ./conf/local.conf
``` 
Comment line with "PACKAGECONFIG:append:pn-qemu-system-native = " sdl" " in ./conf/local.conf 


Now build your linux system;
built images will be put in build/tmp/deploy/images/qemuarmv5
```bash
bitbake freertos-demo #takes about 45 minutes on old laptop
``` 

Configure networking, so that from within Qemu can be communicated
```bash
#
# add networking, virtual interface etc
# thanks to https://bbs.archlinux.org/viewtopic.php?id=207907
sudo ip link add name br0 type bridge
sudo ip addr add 172.20.0.1/16 dev br0
sudo ip link set br0 up
#dnsmasq so that an IP address is assigned dynamically
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
#NB: change wlan0 if host is using another interface for internet access
sudo iptables -A FORWARD -i tap0 -o wlan0 -j ACCEPT
#
#create TAP devices (virtual network adapters for qemu)
sudo runqemu-gen-tapdevs 1000 1000 4 build/tmp/sysroots-components/x86_64/qemu-helper-native/usr/bin/
#
cd build
runqemu nographic
#or to have network connection from qemu
qemu-system-x86_64 ... -net nic,model=virtio -net bridge,br=br0
#to quit qemu enter: ctrl-a followed by 'c', and enter 'quit'
```

