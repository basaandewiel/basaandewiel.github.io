---
layout: post
title: Zephyr Bluetooth programming
---

# Zephyr
Zephyr is an RTOS that should be extremely power efficient and is supported by industry like:
* Nordic
* Intel
* NXP

MBed in contrast is only for ARM, whereas Zephyr also runs on other architectures than ARM.

You can run your Zephyr applications on your x86 machine which might allow you todo a bit of testing on your development machine itself without loading it into the hardware.
This enables for instance developing code for an Arduino_nano_33_ble, and test in on you linux X86 machine (via Qemu). You can even simulate the bluetooth code using BLuez on your Linux machine.


##    Installation
```
sudo apt update
sudo apt upgrade
wget https://apt.kitware.com/kitware-archive.sh
sudo bash kitware-archive.sh
sudo apt install --no-install-recommends git cmake ninja-build gperf   ccache dfu-util device-tree-compiler wget   python3-dev python3-pip python3-setuptools python3-tk python3-wheel xz-utils file   make gcc gcc-multilib g++-multilib libsdl2-dev
cmake --version
python3 --version
dtc --version
pip3 install --user -U west
echo 'export PATH=~/.local/bin:"$PATH"' >> ~/.bashrc
source ~/.bashrc
west init ~/zephyrproject
cd zephyrproject/
west update
west zephyr-export
pip3 install --user -r ~/zephyrproject/zephyr/scripts/requirements.txt
cd
wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.13.2/zephyr-sdk-0.13.2-linux-x86_64-setup.run
chmod +x zephyr-sdk-0.13.2-linux-x86_64-setup.run
./zephyr-sdk-0.13.2-linux-x86_64-setup.run -- -d ~/zephyr-sdk-0.13.2
ls /etc/udev/rules.d/
sudo cp ~/zephyr-sdk-0.13.2/sysroots/x86_64-pokysdk-linux/usr/share/openocd/contrib/60-openocd.rules /etc/udev/rules.d
sudo udevadm control --reload
cd ~/zephyrproject/zephyr
mkdir gnuarmemb
mv Downloads/gcc-arm-none-eabi-9-2019-q4-major-x86_64-linux.tar.bz2 ./
tar -xvf gcc-arm-none-eabi-9-2019-q4-major-x86_64-linux.tar.bz2
rmdir gnuarmemb/
mv gcc-arm-none-eabi-9-2019-q4-major gnuarmemb
export ZEPHYR_TOOLCHAIN_VARIANT=gnuarmemb
```

# Build and run Qemu x86
One of the features I like of Zephyr is that you can test big parts of your embedded software via Qemu on you x86 machine. Togehter with gdb (see below) you can debug on your development machine, without first flashing the firmware to your development board.

To compile and run your software for qemu add `-b qemu_x86`  to the build command line:
```
ZEPHYR_TOOLCHAIN_VARIANT=zephyr
west build --pristine -b qemu_x86 samples/hello_world/
west build -t run
```
The output of printk statements just appear in hour terminal where you started your program.


# Build and run Qemu arm
You can also build the software for another qemu target, like ARM
```
ZEPHYR_TOOLCHAIN_VARIANT=gnuarmemb @@@
ZEPHYR_TOOLCHAIN_VARIANT=zephyr
GNUARMEMB_TOOLCHAIN_PATH=/home/$USER/gnuarmemb/
west build --pristine -b qemu_cortex_m3 samples/hello_world/
#if you get an error, delete `/zepyrproject/zephyr/build directory
west build -t run
```
NB: if you want correct timing for for instance k_sleep calls then following must be added to `prj.conf`
```
CONFIG_QEMU_ICOUNT=n
```

#  Build and run on Nano 33 BLE
To build and flash program for nano 33 BLE, give following commands:
```
GNUARMEMB_TOOLCHAIN_PATH=/home/$USER/gnuarmemb/
ZEPHYR_TOOLCHAIN_VARIANT=gnuarmemb
west build -p auto -b arduino_nano_33_ble samples/bluetooth/peripheral_hr
west flash --bossac=/home/$USER/.arduino15/packages/arduino/tools/bossac/1.9.1-arduino2/bossac
screen /dev/ttyACM0
```
To flash you must use the flash program that is installed when you install the Arduino IDE.

# Build and run on ESP32 (to be tested)
For building software for ESP32 target, you need to install some things:
```
https://docs.zephyrproject.org/latest/boards/xtensa/esp32/doc/index.html
cd ~/zephyrproject
west espressif install
export ESPRESSIF_TOOLCHAIN_PATH="/home/$USER/.espressif/tools/zephyr"
export ZEPHYR_TOOLCHAIN_VARIANT="espressif"
export PATH=$PATH:$ESPRESSIF_TOOLCHAIN_PATH/bin
west espressif update
```

## Build and flash
```
west build --pristine -b esp32 samples/hello_world
west flash
```

## Build for qemu
```
export ESPRESSIF_TOOLCHAIN_PATH="${HOME}/.espressif/tools/zephyr"
#DOES NOT WORK
"${HOME}/.espressif/tools/zephyr/xtensa-esp32-elf"
export ESPRESSIF_TOOLCHAIN_PATH="${HOME}/.espressif/tools/xtensa-esp32-elf/esp-2020r3-8.4.0/xtensa-esp32-elf"
west build --pristine -b qemu_xtensa samples/synchronization
west build -t run
```

# Debugging
In most cases on board probes are used.
gdbstubfeature provides an implementation of the GDB Remote Serial Protocol (RSP) that allows you to **remotely** debug Zephyr using GDB.

This allows us to run the program via Qemu and attach gdb as debugger.


## GNU debugger (gdb)
In order to use gdb, launch QEMU with the -s and-S  options. The-s option will make QEMU listen for an incoming connection from gdb on TCP port 1234, and-S will make QEMU not start the guest until you tell it to from gdb. (If you want to specify which TCP port to use or to use something other than TCP for the gdbstub connection, use the-gdb dev option instead of-s)

First I ran `west --verbose build -t run`. This gives me the total Qemu command that is executed. Then I added the -s and -S options, like this:
`/home/baswi/zephyr-sdk-0.13.2/sysroots/x86_64-pokysdk-linux/usr/bin/qemu-system-i386 -s -S -m 4 -cpu qemu32,+nx,+pae -device isa-debug-exit,iobase=0xf4,iosize=0x04 -no-reboot -nographic -no-acpi -net none -pidfile qemu.pid -chardev stdio,id=con,mux=on -serial chardev:con -mon chardev=con,mode=readline -icount shift=5,align=off,sleep=off -rtc clock=vm -kernel /home/baswi/zephyrproject/zephyr/build/zephyr/zephyr.elf`

Then in another terminal window type:
`gdb ~/zephyrproject/zephyr/build/zephyr/zephyr.elf`
Gdb reads the symbols from the elf file

Now lets gdb connect to QEMU:
 
`(gdb)target remote :1234`

Now you can debug your program with gdb.

* board=arduino_nano_33_ble; 
This can also be run in Qemu; not yet clear to me whether this had advantages over using x86.@@@
Not yet known whether I can use gdb (cmakelist checks boardtype=x86) in debug sample@@@

#### Using Bluetooth and Qemu on x86
```
ZEPHYR_TOOLCHAIN_VARIANT=zephyr
west build --pristine -b qemu_x86 samples/bluetooth/peripheral_hr
west build -t run
```
This gives
`qemu-system-i386: -serial unix:/tmp/bt-server-bredr: **Failed to connect to '/tmp/bt-server-bredr** ':`
Reason is that you have to install a bluetooth stack: BlueZ.


### Install BlueZ
Necessary to be able to run Zephyr programs on x86 which also use Bluetooth
In Linux Mint: 
```
git clone git://git.kernel.org/pub/scm/bluetooth/bluez.git
cd bluez
./bootstrap-configure --disable-android --disable-midi
sudo apt install automake #to install alocal
sudo apt install libtool #elfutils support is required; but elfutils is already installed
sudo apt install elfutils
sudo apt install libdbus-1-dev libudev-dev libical-dev libreadline-dev libjson-c-dev
sudo apt install libbluetooth-dev
sudo apt install libelf-dev
sudo apt-get install libelf-dev elfutils libdw-dev #this solved last elfutils support error 
make
```
gives `ell/util.h: No such file or directory`
does exist, but is a link to an non existing ell directory 1 level up
 
```
apt install libell-dev  #development files for the Embedded Linux library
apt install libell0  #Embedded Linux library
```

gives
`ell/time-private.h - not found`

```
git clone https://git.kernel.org/pub/scm/libs/ell/ell.git
``` 
You’ll need to enable BlueZ’s experimental features so you can access its most recent BLE functionality. Do this by editing the file/lib/systemd/system/bluetooth.service  and making sure to include the-E option in the daemon’s execution start line:
``` 
ExecStart=/usr/libexec/bluetooth/bluetoothd -E
``` 
Finally, reload and restart the daemon:
``` 
sudo systemctl daemon-reload 
sudo systemctl restart bluetooth
```

##### %%%running Qemu Using the Host System Bluetooth Controller

used source: https://docs.zephyrproject.org/latest/guides/bluetooth/bluetooth-tools.html

works via serial port of Qemu -  -serial unix:/tmp/bt-server-bredr
Make sure that the Bluetooth controller is down: 
```
sudo systemctl stop bluetooth
```

Use the btproxy tool to open the listening UNIX socket, type:
```
sudo tools/btproxy -u -i 0
```
gives `Listening on /tmp/bt-server-bredr`

So using the Host System Bluetooth Controller, I use 4 terminal sessions:
* terminal1
    * sudo systemctl stop bluetooth
    * sudo ~/bluez/tools/btproxy -u -i 0

* terminal2
    * west build -b qemu_x86 samples/bluetooth/<sample>
    * west build -t run
    * Now laptop advertises the bluetoothperiphal

* terminal3
    * gdb build/zephyr/zephyr.elf
    * gdb> target remote :5678

* terminal4
    * sudo btmon
    * you can also write output of btmon to file and user Wireshark to analyse messages
        * btmon --write protolog.snoop
 
%%%When I mix the BT heart rate sample with the debug sample, either BT works OR gdb can connect; not both at the same time; **serial socket problem?? Could not find the solution for this after hours of searching** 

  + ALS in prj.conf, BT_CONFIG=n; dan werkt gdb wel
 
CONFIG_GDBSTUB_SERIAL_BACKEND_NAME change from UART_1 to UART_0 in prj.conf

##### interact with Zephyr BT controller
 
sudo ~/bluez/tools/btmgmt --index 0
 
[hci0]#auto-power
 
[hci0]#find -l

* scant for BT devices

##### Ook dit  bestaat
 
arm-none-eabi-gdb

#### aanbevolen: Segger Jlink + JInkGDBserver + GDB
