---
layout: post
title: Syncthing on Raspberry PI
---

Synchting () is a great open source syncing program, that can be used inside your own LAN, without data ever leaving your LAN.
I use it to sync data between different laptops en my Android Phone (it is not supported on IOS, due to high CPU usage).


Recently I installed Syncthing also on my Raspberry pi (running Arch linux). Because my RPI is always on, I can use it to sync from whatever device. When another device is turned on that does not yet contain the latest files, they are synced from RPI to that device,
So with the always on RPI you never miss a sync.

# Installation
```$ sudo pacman -S syncthing```

Install syncthing as system service, so it starts without a user has to login into the RPI.
```
systemctl enable syncthing@<user>.service
systemctl start syncthing@<user>.service
```

Replace <user> by the user name as which the service should run.

# Configuration
Because my RPI has no desktop, I have to make the GUI remote availabe.

* edit `config.xml`  in the syncthing directory
*    change 127.0.0.1 to 0.0.0.0
* on another device (like laptop) point broswer to <IP of RPI>:8384
*    add user name and password to secure the GUI
*    enable https access






