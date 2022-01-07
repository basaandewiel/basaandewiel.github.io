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
```

edit local/conf.local and add/change
```bash
@@@
```
