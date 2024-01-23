---
layout: post
title: Linux LXD container on Arch Linux
---

In this post I describe my experience with installing LXD/Incus containers on Arch Linux.

LXD is an ultra fast linux only hypervisor. LXD containers share the kernel.
So not the whole OS is shared,like Docker.
LXD containers feels like a VM.

Unlike Docker - all data in every container instance is persistent, and any changes you make are permanent unless you revert to a backup. In short, shutting down the container will not erase any issues you might have introduced

The LXD project is no longer part of the Linux Containers project but can now be found directly on Canonical's websites.
A community fork of LXD, Incus, is now part of the Linux Containers project.

On Arch Linux Incus is now the default software for Linux containers.


Installation instructions for Incus on Arch Linux.
```
    pacman -S incus
    systemctl start incus
    incus admin init
```
Answer: 'no bridge'
    define profiles for networking in containers
        source
            https://blog.simos.info/how-to-get-lxd-containers-get-ip-from-the-lan-with-routed-network/
            LXD containers get an IP-address from the LAN (using routed)
        use a routed NIC instead of connecting the instance to the lxdbr0 bridge.
        incus profile create routed
        EDITOR=vim incus profile edit routed
            config:
              user.network-config: |
                version: 2
                ethernets:
                    eth0:
                        addresses:
                        - 192.168.1.30/32
                        nameservers:
                            addresses:
                            - 8.8.8.8
                            search: []
                        routes:
                        -   to: 0.0.0.0/0
                            via: 169.254.0.1
                            on-link: true
            description: Default LXD profile
            devices:
              eth0:
                ipv4.address: 192.168.1.30
                nictype: routed
                parent: end0 ***must match host if***
                type: nic
            name: routed
            used_by: []
        incus profile copy routed routed_192.168.1.30
        evt: incus profile edit routed_192.168.1.30 to adapt IP-addr in it
    create and start container
        incus network list => bride has an IP addr ....200
        incus launch images:archlinux arch1 --profile default --profile routed_192.168.1.30
            incus list => NO IP address
        incus launch images:archlinux arch1
            incus list => IP address 192.168.1.12
            but cannot ping host from within container
            journalctl -u systemd-networkd
        incus  launch images:archlinux arch1 -c security.privileged=true
            incus list => IP address 192.168.1.197
            incus network list => incsbr0 192.168.200/24
            but cannot ping host from within container
            in container # ip route show
                default via 192.168.1.200 dev eth0 proto dhcp src 192.168.1.197 metric 1024
                192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.197 metric 1024
                192.168.1.200 dev eth0 proto dhcp scope link src 192.168.1.197 metric 1024
                but 192.168.1.200 is not pingable
        incus  launch images:archlinux arch1 -c security.privileged=true --profile default --profile routed_192.168.1.30
            incus list => 192.168.1.30 (eth0)
            incus network list =>| incusbr0        | bridge   | YES     | 192.168.1.200/24
            in container: ip route => default via 169.254.0.1 dev eth0
            169.254.0.1 dev eth0 scope link
            can even ping internet URLs
        ip addr a192.168.1.21/24  dev eth0
        ip route add default  via192.168.1.1
    stop/delete container
        incus stop arch1
        incus delete arch1
    host: #ip r
        gives ethernet devices, including those of containers

