---
layout: post
title: Linux LXD/incus container with webserver, external accessible
---

# Introduction
In this post I describe my experience with installing LXD/Incus containers on Arch Linux. 

LXD is an ultra fast linux only hypervisor. LXD containers share the kernel.
So not the whole OS is shared,like with Docker containers.

Unlike Docker - all data in every container instance is persistent, and any changes you make are permanent unless you revert to a backup. In short, shutting down the container will not erase any issues you might have introduced.
This means that you can for instance start an interactive shell in the container, install some packages and then stop the container. When you restart the container the packages you just installed are still present.
So the container feels like a VM.

The LXD project is no longer part of the Linux Containers project but can now be found directly on Canonical's websites.
A community fork of LXD, Incus, is now part of the Linux Containers project.

On Arch Linux Incus is now the default software for Linux containers. This is also the reason I used `incus` in stead of LXD.


## Used sources
* https://blog.simos.info/how-to-get-lxd-containers-get-ip-from-the-lan-with-routed-network/

# Installation
The installion of incus is straightforward. For me the most difficult part was to get internet access from within the running container, so the networking part.


Install and init incus.
```
    pacman -S incus
    systemctl start incus
    incus admin init
```
Now you get an amount of questions you have to answer. For most questions the default value is good.
For the networking part, answer that you **do not need a bridge**.
We use a routed NIC instead of connecting the instance to the lxdbr0 bridge.
For simplicity we assign a static IP address to the container. Because we can use an address that belongs to the LAN the host is on, the web server in the container will be easily accesble from outside.

```
incus profile create routed_192.168.1.30
EDITOR=vim incus profile edit routed
```
and insert the following is this profile:
```
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
        parent: end0
        type: nic
    name: routed_192.168.1.30
    used_by: []
```

**NB: the parent above, must match the device name of the host**

You have a course to adapt the above IP address to fit in your LAN; and use an IP-address outside the range of you DHCP-server.


Now create and launch an arch linux container named `arch1`.
```
incus  launch images:archlinux arch1 -c security.privileged=true --profile default --profile routed_192.168.1.30
```
This line first applies the default profile, and then the routed profile on top.
security.privileged is necessary to make the container accessible from the outside.
With the command `incus list` you can see what IP-address to assigned to container arch1.

To start an interactive shell in the container `arch1`, use following command:
```
incus exec arch1 -- bash
```

## No internet access from within the container
A problem that occurs regulary is that there is no internet access from within the container.
Note that if your system supports and uses nftables, LXD detects this during installation and switches to nftables mode. In this mode, LXD adds its rules into the nftables, using its own nftables namespace.


If you do not have internet access then go back to the host system, and try following commands
```
nft flush ruleset
systemctl restart incus
```
This fushes the rules of the firewall nft, and restarts incus. Especially if you also use Docker on the host system, docker adds some rules to the nft firewall. If you have internet access in the container after flushing the nft firewall rules, you know where to search the cause of the problem. Most times is has to do with the FORWARD chain, where packets are not accepted.
The order of starting Incus, own fw and Docker can influence what fw-rules are active.


Now can install packages etc, which are persistant after stopping the container. So you can also easily install a webserver, like nginx.
```
incus exec arch1 -- bash
# pacman -Suy
# pacman -S nginx
```
Now you can configure a website in nginx, and add the file to /srv/http/

To stop the container:
```
incus stop arch1
```

