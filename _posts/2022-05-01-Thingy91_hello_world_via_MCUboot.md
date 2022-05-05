---
layout: post
title: Nordic Thingy91 - hello world, built with VScode, programmed via USB (MCUboot)
---

This posts summarises how you can run the hello world sample on the Nordic Thingy91. I had to do some special things because I do not have a programmer. So I had to program the hello word sample via MCUboot.

NB: I assume that MCUboot is already programmed on your Thingy91, and that you have installed all necessary software as nRF Connect for Desktop (see Nordic documentation on how to do this)

* VScode
    * create build configuration
        * set board to thingy91_nrf9160_ns (so no secure version!). By doing this CONFIG_BOOTLOADER_MCUBOOT is set to 'y' (via `thingy91_nrf9160_ns_defconfig`), so we do not have to do this in our prj.conf file.
Without this CONFIG_BOOTLOADER_MCUBOOT setting the necessary hex file `app_signed.hex` is not built; 
    * build the application
    * close the serial connection in vscode, or close vscode (otherwise you can not program hex file to Thingy, because the com port is in use)
* put thing91 in mode for programming nrf9160 (switch Thingy91 off; hold big middle button and switch it on again)
* Open NCS->programmer app via nRF Connect for Desktop
    * select device
    * enable MCUboot
    * clear files
    * add file; browse to `...samples/helloworld/build/zephyr/app_signed.hex`
    * write file to thingy
* Thingy: switch off and on via switch (plugging in/out USB cable is not enough, because Thingy91 is also battery powered)
* start VScode (I used VScode to connect to the COM port; I could not get this working via Putty on Windows yet)
    * nrf terminal; connect to COM6 (windows), 115200 8n2 rtscts:off
* In linux the following also works: `screen /dev/ttyACM0 115200`
