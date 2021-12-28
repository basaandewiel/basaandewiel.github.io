---
layout: post
title: OpenWRT - configure
---
## Configure adblock
Adblock is easily installed via the GUI.
To confiugre whitelisting follwwing gmust be done

Create file `/etc/adblock/adblock.whitelist` with following contents (if you want to be able to click on Google advertisements)
`www.googleadservices.com`

After this restart adblock
`/etc/init.d/adblock restart`
 
