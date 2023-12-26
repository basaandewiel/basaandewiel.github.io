---
layout: post
title: Nordic Thingy91 - hello world, built with VScode, programmed via USB (MCUboot)
---

This posts summarises how you can run the hello world sample on the Nordic Thingy91. I had to do some special things because I do not have a programmer. So I had to program the hello word sample via MCUboot.

# Installing
I assume that MCUboot is already programmed on your Thingy91.

First you have to install all necessary software as nRF Connect for Desktop (see Nordic documentation https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/gs_assistant.html)


## Easiest way - GUI
The easiest way should be to 
* install "nRF connect for Desktop"
* from within "nRF connect for Deskop" install "Toolchain manager"
* from within "Toolchain manager" install "nRF connect SDK"
* from the latter you should be able to configure VScode with the necessary plugins

On Ubuntu 20.0 I could not get everything working; VScode does work with the plugins, but complaints that it cannot find the "Nrf command line tools"  unless they are installed.
I ended up with 
* Using VScode for developing the application, only.
* Flashing is done via "nRF Connect for Desktop" and then starting the "Programmer"; see below.
* Installing Connect for Desktop
    * Download the Appimage for linux; could not install version 3.12 (gave error about graphics driver); Solution was to install version 3.9, and let it upgrade itself

## Install via command line (Ubuntu)
Installing via the command line is also easy, at least if you know the exact commands. These are summarized below:
```
sudo apt update
sudo apt upgrade

wget https://apt.kitware.com/kitware-archive.sh
sudo bash kitware-archive.sh
sudo apt install --no-install-recommends git cmake ninja-build gperf   ccache dfu-util device-tree-compiler wget   python3-dev python3-pip python3-setuptools python3-tk python3-wheel xz-utils file   make gcc gcc-multilib g++-multilib libsdl2-dev
        
pip3 install west
          
west init -m https://github.com/nrfconnect/sdk-nrf /home/runner/work
cd /home/runner/work
west update
pip3 install -r zephyr/scripts/requirements-base.txt
                    
sudo apt install --no-install-recommends cmake ninja-build
          
#install toolchains
cd /usr/local/
# get minimal version, without toolchains
sudo wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.15.1/zephyr-sdk-0.15.1_linux-x86_64_minimal.tar.gz
sudo tar -xvf zephyr-sdk-0.15.1_linux-x86_64_minimal.tar.gz
          
# install only necessary toolchains
cd zephyr-sdk-0.15.1
sudo wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.15.1/toolchain_linux-x86_64_arm-zephyr-eabi.tar.gz
sudo tar -xvf toolchain_linux-x86_64_arm-zephyr-eabi.tar.gz
sudo wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.15.1/toolchain_linux-x86_64_x86_64-zephyr-elf.tar.gz
sudo tar -xvf toolchain_linux-x86_64_x86_64-zephyr-elf.tar.gz

export ZEPHYR_TOOLCHAIN_VARIANT=zephyr
export ZEPHYR_SDK_INSTALL_DIR=/opt/toolchains/zephyr-sdk-0.15.1/
```


# Build and flash hello world app
* edit ...samples/helloworld/prj.conf
    * add line with "CONFIG_BOOTLOADER_MCUBOOT=y" //so that correct hex file is build

* VScode
    * create build configuration
        * set board to thingy91_nrf9160_ns (so no secure version!). By doing this CONFIG_BOOTLOADER_MCUBOOT is set to 'y' (via `thingy91_nrf9160_ns_defconfig`), so we do not have to do this in our prj.conf file.
Without this CONFIG_BOOTLOADER_MCUBOOT setting the necessary hex file `app_signed.hex` is not built; 
    * build the application
```
          pip install cbor2
          west build --pristine -b thingy91_nrf9160_ns
```
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

