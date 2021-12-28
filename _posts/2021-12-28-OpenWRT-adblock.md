---
layout: post
title: OpenWRT - configure
---
## Configure adblock
Adblock is easily installed via the GUI.
To confiugre whitelisting follwwing gmust be done

Create file `/etc/adblock/adblock.whitelist`.
Put all the URLs in this file, which you want to be white listed. So if you for instance want to be able to click on Google advertisements), you have to add following contents to this file:
`www.googleadservices.com`

To take effect, you have to  restart adblock:
`/etc/init.d/adblock restart`
 
