---
layout: post
title: Arch linux on new Raspberry Pi 5B
---

Although Arch linux is not (yet) officially released for the new Raspberry Pi 5B, I wanted to get it working (because I am a little bit addicted to arch linux).
I found an good site that describes how to do this, so all credits go to this site.

https://kiljan.org/2023/11/24/arch-linux-arm-on-a-raspberry-pi-5-model-b/ by 
© 2023Sven Kiljandd

This post is mainly for myself to describe what I did.

# Introduction

U-Boot does not support the Raspberry Pi 5. In turn, Arch Linux ARM does not support the Raspberry Pi 5 since the Pi 5’s CPU cores only support booting in 64-bit mode and Arch Linux ARM’s aarch64 Raspberry Pi image relies on U-Boot.

It is possible to get Arch Linux ARM up and running on a Raspberry Pi 5 by removing U-Boot and replacing the mainline kernel with a directly booting kernel from the Raspberry Pi foundation. Automatic updates will even work since the replacement kernel is available as an Arch Linux ARM package. Sweet!


# Prepare the microSD card

This mostly concerns following the official instructions from Arch Linux ARM for older Raspberry Pi models.
Copy Arch Linux ARM to the microSD card

The following commands concern the installation of Arch Linux ARM to a microSD card, conforming to the official installation instructions.

For these commands, it is assumed that your SD card reader is offered as device /dev/mmcblk0. Adjust the configuration variables if necessary, for example when a USB SD card reader is used. Be careful to not overwrite data on a device you do not want to write to!

You can adjust the download directory in case RAM is limited on the computer used to prepare the microSD card.

NB: make sure that any partitions present on the sdcard are NOT MOUNTED

Run as root:
'''
export SDDEV=/dev/mmcblk0
export SDPARTBOOT=/dev/mmcblk0p1
export SDPARTROOT=/dev/mmcblk0p2
export SDMOUNT=/mnt/sd
export DOWNLOADDIR=/tmp/pi
export DISTURL="http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-aarch64-latest.tar.gz"

mkdir -p $DOWNLOADDIR
(
  cd $DOWNLOADDIR && \
  curl -JLO $DISTURL
)

sfdisk --quiet --wipe always $SDDEV << EOF
,256M,0c,
,,,
EOF

mkfs.vfat -F 32 $SDPARTBOOT
mkfs.ext4 -E lazy_itable_init=0,lazy_journal_init=0 -F $SDPARTROOT

mkdir -p $SDMOUNT
mount $SDPARTROOT $SDMOUNT
mkdir -p ${SDMOUNT}/boot
mount $SDPARTBOOT ${SDMOUNT}/boot

bsdtar -xpf ${DOWNLOADDIR}/ArchLinuxARM-rpi-aarch64-latest.tar.gz -C $SDMOUNT

Remove U-Boot and the mainline kernel manually

This is removed in the most barbaric way, but it works! Make sure to run this command in the same context as the previous commands as root:

rm -rf ${SDMOUNT}/boot/*

Add the Pi Foundation kernel

mkdir -p ${DOWNLOADDIR}/linux-rpi
pushd ${DOWNLOADDIR}/linux-rpi
curl -JLO http://mirror.archlinuxarm.org/aarch64/core/linux-rpi-6.1.63-1-aarch64.pkg.tar.xz
tar xf *
cp -rf boot/* ${SDMOUNT}/boot/
popd
''' 

You’ll probably need to replace the download link with the one for the latest package version, which can be found on this@@@https://archlinuxarm.org/packages/aarch64/linux-rpi@@@ page.
Remove the SD card safely

Run as root:
'''
sync && umount -R $SDMOUNT
'''

NB: the sync command can take up to 5 minutes, if you have a slow sd card.

Now remove the microSD card.
Boot the Raspberry Pi to update Arch Linux ARM

Insert the microSD card in the Raspberry Pi, connect it to a wired network with internet access, and apply power.
Login

Get access to the Pi using an HDMI display and a USB keyboard, or use a wired network connection (configured for DHCP by default) with SSH or a serial connection.

Use the Arch Linux ARM default credentials.

Username: alarm Password: alarm

After login, use su root to gain root access. The password of the root account is ‘root’.

All following commands are executed on the Raspberry Pi as root.
Update Arch Linux ARM

In previous steps, the boot loader was forcibly removed and the kernel forcibly replaced. This ‘dirty state’ of manually replaced files does not reflect the currently installed Arch Linux ARM packages. This must be corrected by first removing the packages of the files that are not there anymore. Next, the package for the currently used kernel must be installed.
'''
pacman-key --init
pacman-key --populate archlinuxarm

pacman -R linux-aarch64 uboot-raspberrypi
pacman -Syu --overwrite "/boot/*" linux-rpi
'''

Reboot the Pi

Now do a reboot:

'''reboot'''

When the Pi arrives at the user login prompt, the HDMI video signal will switch to a higher resolution. If a supported fan is installed on the Pi’s fan header, it will now be controlled instead of running at maximum speed. Once logged in, you’ll notice with ip addr that the wireless network adapter is available.

That’s it. You now have basic Arch Linux ARM running on your Pi 5. Start customizing it to your needs!
Updating

While it is true that Arch Linux ARM does not support installation on Raspberry Pi 5 yet, effectively it supports running on it since all required packages are available in the official repositories. This includes the latest firmware files. These files are needed to support functions like as Wi-Fi and Bluetooth, and were not mentioned earlier in this guide since it was not needed. They were upgraded when pacman -Syu was executed.

Running pacman -Syu in the future will install also install the latest Pi Foundation kernel if an update is available. Since current and newer versions of the installed packages do support the Pi 5, it will continue to work without issue with up-to-date software using only packages provided by Arch Linux ARM.
Handling future official Pi 5 support

When Arch Linux ARM starts supporting the Pi 5, the Pi Foundation’s kernel can be replaced with the mainline kernel by running:
'''
pacman -Syu linux-aarch64 uboot-raspberrypi
'''

There will be warning that those packages conflict with package linux-rpi and whether you want it replaced. If you do, linux-rpi will be removed before installing the new packages. After that, your Arch Linux ARM installation should be the same as the official Arch Linux ARM Raspberry Pi image that supports the Pi 5.

    raspberry-pi
    raspberry-pi-5
    arch-linux-arm
    aarch64

© 2023Sven Kiljandd


