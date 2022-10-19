---
layout: post
title: Zephyr programming with Arduino nano 33 BLE
---
I was looking for an RTOS to program my Arduino nano 33 BLE. First I used Arduino/mbed, but later I came across Zephyr. Whereas mbed is only supported for ARM architectures, Zephyr also supports other architectures like ESP32/Tensilica. This enables reuse of your programs on different architectures.


Even x86 architectures are supported by Zephyr, and this enables very interesting possibilities of running/debugging programs on the development host in stead of on the target. This is especially interesting for developing BLE applications because of BLE monitoring tools that can run on the host together with the application that is developed. 
This enables for instance developing BLE applications for an Arduino_nano_33_ble, and test it on your linux X86 machine (via Qemu). You can even simulate the bluetooth code using BLuez on your Linux develop machine.



## Installation
You can find good documentation on installing Zephyr. Below are the instructions I used to install Zephyr on Linux mint 20.3.

NB: On Linux installing via the tools SDK manager and Toolchain manager, and automatically installing required VScode extensions, does give an error in VScode: `could not find nrfjprog` where that tool is in the PATH.
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
```
download from website: zephyr-sdk-0.15.1_linux-aarch64_minimal.tar.gz 

```
cd ~/Downloads
tar -xvf zephyr-sdk-0.15.1_linux-aarch64_minimal.tar.gz 
cd zephyr-sdk-0.15.1/
./setup.sh 
cd ..
mv zephyr-sdk-0.15.1 /usr/local
sudo mv zephyr-sdk-0.15.1 /usr/local
cd /usr/local
chmod 777 zephyr-sdk-0.15.1/
cd zephyr-sdk-0.15.1/
./setup.sh 

```

The instructions below were not executed after the failing install via SDK manager and Toolchain manager.
```
ls /etc/udev/rules.d/
sudo cp ~/zephyr-sdk-0.13.2/sysroots/x86_64-pokysdk-linux/usr/share/openocd/contrib/60-openocd.rules /etc/udev/rules.d
sudo udevadm control --reload
cd ~/zephyrproject/zephyr
mv ~/Downloads/gcc-arm-none-eabi-9-2019-q4-major-x86_64-linux.tar.bz2 ./
tar -xvf gcc-arm-none-eabi-9-2019-q4-major-x86_64-linux.tar.bz2
mv gcc-arm-none-eabi-9-2019-q4-major gnuarmemb
export ZEPHYR_TOOLCHAIN_VARIANT=gnuarmemb
```

## Build and run Qemu x86
One of the features I like of Zephyr is that you can test big parts of your embedded software via Qemu on your x86 machine. Togehter with gdb (see below) you can debug on your development machine, without first flashing the firmware to your development board.

To compile and run your software for qemu add `-b qemu_x86`  to the build command line:
```
ZEPHYR_TOOLCHAIN_VARIANT=zephyr
west build --pristine -b qemu_x86 samples/hello_world/
west build -t run
```
The output of printk statements just appear in your terminal where you started your program.


## Build and run Qemu arm
You can also build the software for another qemu target, like ARM
```
ZEPHYR_TOOLCHAIN_VARIANT=gnuarmemb 
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

##  Build and run on Nano 33 BLE
To build and flash a program for Arduino nano 33 BLE, give following commands:
```
GNUARMEMB_TOOLCHAIN_PATH=/home/$USER/gnuarmemb/
ZEPHYR_TOOLCHAIN_VARIANT=gnuarmemb
west build -p auto -b arduino_nano_33_ble samples/bluetooth/peripheral_hr
west flash --bossac=/home/$USER/.arduino15/packages/arduino/tools/bossac/1.9.1-arduino2/bossac
screen /dev/ttyACM0
```
To flash you must use the flash program that is installed when you install the Arduino IDE.

One problem is that printk statements do not show up. This is caused by the fact that after flahsing the device /dev/ttyACM0 is gone. This can be solved as follows:

Create a file named `app.overlay` in the application directory (not in the src directory). This file should have the following contents:
```
/ {
	chosen {
		zephyr,console = &cdc_acm_uart0;
	};
};

&zephyr_udc0 {
	cdc_acm_uart0: cdc_acm_uart0 {
		compatible = "zephyr,cdc-acm-uart";
		label = "CDC_ACM_0";
	};
};
```

In your program you have to add following lines:
```
	const struct device *dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_console));
	uint32_t dtr = 0;

	if (usb_enable(NULL)) {
		return;
	}

	/* Poll if the DTR flag was set */
	while (!dtr) {
		uart_line_ctrl_get(dev, UART_LINE_CTRL_DTR, &dtr);
		/* Give CPU resources to low priority threads. */
		k_sleep(K_MSEC(100));
	}
```
The file prj.conf must have following contents:
```
CONFIG_USB_DEVICE_STACK=y
CONFIG_USB_DEVICE_PRODUCT="Program name, free to choose"

CONFIG_SERIAL=y
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y
CONFIG_UART_LINE_CTRL=y
```

And the file Kconfig must have following contents:
```
source "Kconfig.zephyr"
```
After building your program again printk statements should work.
To see the printk output you have to connect a terminal program to the right device, like
`screen /dev/ttyACM0`. To quit the screen command use ctrl-A followed by 'd'.
NB: when 'screen'  is not installed, you have to install it first :).

## Build and run Zephyr on ESP32 (to be tested)
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

### Build and flash
```
west build --pristine -b esp32 samples/hello_world
west flash
```

### Build for qemu
```
export ESPRESSIF_TOOLCHAIN_PATH="${HOME}/.espressif/tools/zephyr"
#DOES NOT WORK
"${HOME}/.espressif/tools/zephyr/xtensa-esp32-elf"
export ESPRESSIF_TOOLCHAIN_PATH="${HOME}/.espressif/tools/xtensa-esp32-elf/esp-2020r3-8.4.0/xtensa-esp32-elf"
west build --pristine -b qemu_xtensa samples/synchronization
west build -t run
```

## Debugging
In most cases on board probes are used.
The gdbstub feature provides an implementation of the GDB Remote Serial Protocol (RSP) that allows you to debug Zephyr running on your development machine using ```gdb```.

This allows us to run the program via Qemu and attach gdb as debugger.


### GNU debugger (gdb)
In order to use gdb, you have to launch QEMU with the -s and-S options. The-s option will make QEMU listen for an incoming connection from gdb on TCP port 1234, and-S will make QEMU not start the guest until you tell it to from gdb. (If you want to specify which TCP port to use or to use something other than TCP for the gdbstub connection, use the ```-gdb dev``` option instead of-s)

First I ran `west --verbose build -t run`. This gives me the total Qemu command that is executed. Then I added the -s and -S options, like this:

`/home/$USER/zephyr-sdk-0.13.2/sysroots/x86_64-pokysdk-linux/usr/bin/qemu-system-i386 -s -S -m 4 -cpu qemu32,+nx,+pae -device isa-debug-exit,iobase=0xf4,iosize=0x04 -no-reboot -nographic -no-acpi -net none -pidfile qemu.pid -chardev stdio,id=con,mux=on -serial chardev:con -mon chardev=con,mode=readline -icount shift=5,align=off,sleep=off -rtc clock=vm -kernel /home/baswi/zephyrproject/zephyr/build/zephyr/zephyr.elf`

Then in another terminal window type:

`gdb ~/zephyrproject/zephyr/build/zephyr/zephyr.elf`

Gdb reads the symbols from the elf file

Now let gdb connect to QEMU:
 
`(gdb)target remote :1234`

Now you can debug your program with gdb.

<!--
* board=arduino_nano_33_ble; 
Now we want to run bluetooth applications on our host machine.
This can also be run in Qemu; not yet clear to me whether this had advantages over using x86.@@@
Not yet known whether I can use gdb (cmakelist checks boardtype=x86) in debug sample@@@
-->

#### Using Bluetooth and Qemu on x86
Now we want to run bluetooth applications on our host machine.
```
ZEPHYR_TOOLCHAIN_VARIANT=zephyr
west build --pristine -b qemu_x86 samples/bluetooth/peripheral_hr
west build -t run
```
This gives
`qemu-system-i386: -serial unix:/tmp/bt-server-bredr: **Failed to connect to '/tmp/bt-server-bredr** ':`
Reason is that you first have to install a bluetooth stack: BlueZ.


### Install BlueZ
This is necessary to be able to run Zephyr programs on x86 which also use Bluetooth.

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

##### Running Qemu Using the Host System Bluetooth Controller

Used source: https://docs.zephyrproject.org/latest/guides/bluetooth/bluetooth-tools.html

Bluez works via serial port of Qemu -  -serial unix:/tmp/bt-server-bredr
This only works if the Bluetooth controller is down: 
```
sudo systemctl stop bluetooth
```

Use the btproxy tool to open the listening UNIX socket, type:
```
sudo tools/btproxy -u -i 0
```
gives `Listening on /tmp/bt-server-bredr`

So for using the Host System Bluetooth Controller, I use 4 terminal sessions:
* terminal-1
    * ```sudo systemctl stop bluetooth```
    * ```sudo ~/bluez/tools/btproxy -u -i 0```

* terminal-2
    * ```west build -b qemu_x86 samples/bluetooth/<sample>```
    * ```west build -t run```
    * Now the laptop advertises the bluetoothperiphal

* terminal-3
    * ```gdb build/zephyr/zephyr.elf```
    * ```gdb> target remote :5678```

* terminal-4
    * ```sudo btmon```
    * you can also write output of btmon to file and use Wireshark to analyse messages
        * btmon --write protolog.snoop
 
When I mix the BT heart rate sample with the debug sample, either BT works OR gdb can connect; not both at the same time; **serial socket problem?? Could not find the solution for this after hours of searching** 

 
CONFIG_GDBSTUB_SERIAL_BACKEND_NAME change from UART_1 to UART_0 in prj.conf

##### interact with Zephyr BT controller
 
sudo ~/bluez/tools/btmgmt --index 0
 
[hci0]#auto-power
 
[hci0]#find -l

* scans for BT devices

<!-- Ook dit  bestaat
arm-none-eabi-gdb

#### aanbevolen: Segger Jlink + JInkGDBserver + GDB
-->
