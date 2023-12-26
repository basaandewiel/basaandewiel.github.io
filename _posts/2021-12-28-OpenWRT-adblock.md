---
layout: post
title: OpenWRT - configure
---
## Install adblock
You have to install adblock itself and a luci-helper to be able to configure adblock via the GUI:
```bash
opkg update
opkg install adblock
opkg install luci-app-adblock
```

## Configure adblock
Adblock is easily installed via the GUI.
To confiugre whitelisting follwwing gmust be done

Create file `/etc/adblock/adblock.whitelist`.
Put all the URLs in this file, which you want to be white listed. So if you for instance want to be able to click on Google advertisements), you have to add following contents to this file:
`www.googleadservices.com`

To take effect, you have to  restart adblock:
`/etc/init.d/adblock restart`
 
 NB: if the adblock definitions cannot be downloaded (as you can see in the log view), then try
 `opkg update; opkg install libcurl4` (worked on OpenWRT 21.02.1 r16325-88151b8303 / LuCI openwrt-21.02 branch git-21.295.67054-13df80d)
