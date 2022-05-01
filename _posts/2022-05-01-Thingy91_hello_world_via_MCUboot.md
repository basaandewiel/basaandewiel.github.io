---
layout: post
title: Nordic Thingy91 - hello world, built with VScode, programmed via USB (MCUboot)
---

This posts summarises how you can run the hello world sample on the Nordic Thingy91. I had to do some special things because I do not have a programmer. So I had to program the hello word sample via MCUboot.

* edit `...samples/hello_world/prj.conf`
*    add line with `CONFIG_BOOTLOADER_MCUBOOT=y` //so that correct hex file is built
* VScode
*    create build configuration
        set board to thingy91_nrf9160_ns (so no secure version)
    build
    close vscode (otherwise you can not program hex file to thingy)
put thing91 in mode for programming nrf9160 (switch of; hold big middle button and switch on)
open NCS->programmer app
    select device
    enable MCUboot
    clear files
    add file; browse to ...samples/helloworld/build/zephyr/app_signed.hex
    write file to thingy
thingy: switch off and on via switch
VScode (or putty)
    nrf terminal; connect to COM6 (windows), 115200 8n2 rtscts:off

