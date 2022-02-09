# Zephyr
Zephyr is the future and is extremely power efficient. 
    supported by industry
* Nordic
* Intel
* NXP
* MBed is only for ARM, Zephyr also runs on other architectures.

You can run your Zephyr applications on your x86 machine which might allow you todo a bit of testing on your development machine itself without loading it into the hardware.
This enable for instance developing code for an Arduino_nano_33_ble, and test in on you linux X86 machine (via Qemu). You can even simulate the bluetooth code using BLuez on your Linux machine.


##    installation
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

# build and run Qemu x86
```
ZEPHYR_TOOLCHAIN_VARIANT=zephyr
west build --pristine -b qemu_x86 samples/hello_world/
west build -t run
`

# build and run Qemu arm
```
ZEPHYR_TOOLCHAIN_VARIANT=gnuarmemb
ZEPHYR_TOOLCHAIN_VARIANT=zephyr
GNUARMEMB_TOOLCHAIN_PATH=/home/$USER/gnuarmemb/
west build --pristine -b qemu_cortex_m3 samples/hello_world/
#if you get an error, delete `/zepyrproject/zephyr/build directory
west build -t run
`

#  build and run on Nano 33 BLE
```
GNUARMEMB_TOOLCHAIN_PATH=/home/$USER/gnuarmemb/
ZEPHYR_TOOLCHAIN_VARIANT=gnuarmemb
west build -p auto -b arduino_nano_33_ble samples/bluetooth/peripheral_hr
west flash --bossac=/home/$USER/.arduino15/packages/arduino/tools/bossac/1.9.1-arduino2/bossac
screen /dev/ttyACM0
`

# ESP32
```https://docs.zephyrproject.org/latest/boards/xtensa/esp32/doc/index.html
cd ~/zephyrproject
west espressif install
export ESPRESSIF_TOOLCHAIN_PATH="/home/$USER/.espressif/tools/zephyr"
export ZEPHYR_TOOLCHAIN_VARIANT="espressif"
export PATH=$PATH:$ESPRESSIF_TOOLCHAIN_PATH/bin
west espressif update
`
#build and flash
```
west build --pristine -b esp32 samples/hello_world
west flash
#NOT YET TESTED
`

#build en qemu
export ESPRESSIF_TOOLCHAIN_PATH="${HOME}/.espressif/tools/zephyr"
#DOES NOT WORK
"${HOME}/.espressif/tools/zephyr/xtensa-esp32-elf"
export ESPRESSIF_TOOLCHAIN_PATH="${HOME}/.espressif/tools/xtensa-esp32-elf/esp-2020r3-8.4.0/xtensa-esp32-elf"
west build --pristine -b qemu_xtensa samples/synchronization
west build -t run
`

# Debugging
In most cases on board probes are used.
gdbstubfeature provides an implementation of the GDB Remote Serial Protocol (RSP) that allows you to remotely debug Zephyr using GDB.

## Qemu
Mostly it is not used as emulatorbut as virtualizerin collaboration with KVM kernelcomponents. In that case it utilizes the virtualization technology of the hardware to virtualize guests.
            consists of
                CLI
                Monitor
            debugging
                you can attach any debugger  that use GDB remote protocol
                    GNU debugger

In order to use gdb, launch QEMU with the -s and -S  options. The -s option will make QEMU listen for an incoming connection from gdb on TCP port 1234, and -S will make QEMU not start the guest until you tell it to from gdb. (If you want to specify which TCP port to use or to use something other than TCP for the gdbstub connection, use the -gdb dev option instead of -s. See Using unix sockets for an example.)
                 <https://qemu-project.gitlab.io/qemu/system/gdb.html#using-unix-sockets>
                In gdb, connect to QEMU:
                    (gdb) target remote localhost:1234
                board=qemu_x86; runs in qemu, debug with gdb
                    kan ik program flow mee debuggen
                board=arduino_nano_33_ble; kan ik ook met qemu runnen
                    heeft dit nut tov qemu_x86
                    weet niet of ik dat ook kan gdb-en (cmakelist checks boardtype=x86) in debug sample
                als ik BT program run in q
                    ZEPHYR_TOOLCHAIN_VARIANT=zephyr
                    west build --pristine -b qemu_x86 samples/bluetooth/peripheral_hr
                    west build -t run
                        qemu-system-i386: -serial unix:/tmp/bt-server-bredr: Failed to connect to '/tmp/bt-server-bredr':
                    install
                        BlueZ needed
                            standard in linux mint (is not a command)
                        blueztools
                            git clone git://git.kernel.org/pub/scm/bluetooth/bluez.git
                            cd bluez
                            ./bootstrap-configure --disable-android --disable-midi
                                sudo apt install automake
                                    to install alocal
                                sudo apt install libtool
                                elfutils support is required
                                    but elfutils is already installed
                                sudo apt install elfutils
                                sudo apt install libdbus-1-dev libudev-dev libical-dev libreadline-dev libjson-c-dev
                                sudo apt install libbluetooth-dev
                                sudo apt install libelf-dev
                                sudo apt-get install libelf-dev elfutils libdw-dev
                                    deze loste laatste elfutils support error op
                            make
                                ell/util.h: No such file or directory
                                    bestaat wel maar is een link naar niet bestaande ell directory 1 nivo hoger
                                    libell-dev installed
                                     <https://packages.ubuntu.com/focal/libell-dev>
                                        development files for the Embedded Linux library
                                    libell0 installed
                                     <https://packages.ubuntu.com/focal/libell0>
                                        Embedded Linux library
                                nu: ell/time-private.h - not found
                                    git clone https://git.kernel.org/pub/scm/libs/ell/ell.git
                        You’ll need to enable BlueZ’s experimental features so you can access its most recent BLE functionality. Do this by editing the file /lib/systemd/system/bluetooth.service  and making sure to include the -E option in the daemon’s execution start line:
                            ExecStart=/usr/libexec/bluetooth/bluetoothd -E
                        Finally, reload and restart the daemon:
                            sudo systemctl daemon-reload
                            sudo systemctl restart bluetooth
                    running Qemu Using the Host System Bluetooth Controller
                        https://docs.zephyrproject.org/latest/guides/bluetooth/bluetooth-tools.html
                        works via serial port of Qemu -  -serial unix:/tmp/bt-server-bredr
                        Make sure that the Bluetooth controller is down: sudo systemctl stop bluetooth
                        Use the btproxy tool to open the listening UNIX socket, type:
                            sudo tools/btproxy -u -i 0
                            Listening on /tmp/bt-server-bredr
                    So,
                        Using the Host System Bluetooth Controller
                            terminal1
                                sudo systemctl stop bluetooth
                                sudo ~/bluez/tools/btproxy -u -i 0
                            terminal2
                                west build -b qemu_x86 samples/bluetooth/<sample>
                                west build -t run
                            Now laptop advertises the bluetoothperiphal
                            terminal3
                                gdb build/zephyr/zephyr.elf
                                gdb> target remote :5678
                            terminal4
                                sudo btmon
                            When I mix the BT heart rate sample with the debug sample, either BT works OR gdb can connect; not both at the same time; serial socket problem?? Could not find the solution for this after hours of searching
                                ALS in prj.conf, BT_CONFIG=n; dan werkt gdb wel
                            CONFIG_GDBSTUB_SERIAL_BACKEND_NAME change from UART_1 to UART_0 in prj.conf
                    interact with Zephyr BT controller
                        sudo ~/bluez/tools/btmgmt --index 0
                        [hci0]# auto-power
                        [hci0]# find -l
                            scant for BT devices
                    Ook dit  bestaat
                        arm-none-eabi-gdb
                aanbevolen: Segger Jlink + JInkGDBserver + GDB
    config files
        CMakeLists.txt:
            top-level file for the CMake build system,
            *
            This file tells the build system where to find the other application files, and linksthe application directory with Zephyr’s CMake build system. This link provides features supported by Zephyr’s build system, such as board-specific kernel configuration files, the ability to run and debug compiled binaries on real or emulated hardware, and more.
        Kconfig (file prj.conf)
            refers to Kconfig.zephyr
            *
            Kernel configuration files: An application typically provides a Kconfig  configuration file (usually called prj.conf) that specifies application-specific values for one or more kernel configuration options. These application settings are merged with board-specific settings to produce a kernel configuration.
                bv of je GPS of MQTT nodig hebt
                CONFIG_CPLUSPLUS=y
        west.yml
            manifest
            external repositories
            mulit repositiory management (bepaalt welke versies vd versch repositroies die je gebruikt, in een bepaalde versie van jouw product zit)
        dts
            device tree
            describes non-discoverable board-specific hardware details
    ninja = build tool
    toolchain
        defined in ~/.zephyrrc
        je hebt ook ~/zephyrproject/zephyr/zephyr_env.sh
    create new app
        cd ~/zephyrproject
        source ~/zephyrproject/zephyr/zephyr_env.sh
        west update
    zephyr.elf image contains both the applicationand the kernel libraries.
    nRF connect SDK
        uses Zephyr
        There’s value in using the nRF SDK for a more bare metalimplementation
        NB: nRF5 SDK is NOT an RTOS, this section covers "nRF Connect SDK"
        MCUboot
            open source bootloader
        802.15.4
        Toolchain
            Kconfig+Devicetree => CMake => Ninja => GCC
            Kconfig
                Kconfig configurations are stored in configuration files. The initial configurationis generated by mergingconfiguration fragments from the board and application(e.g. prj.conf).
            DeviceTree
                describes hardware
                possible to support different hardware without changing source code
            Cmake
                generaties build files
            Ninja
                similiar to make, but faster for incremental build
                needs Cmake
        Bluetooth soft device = BT protocol stack -deveoped by Nordic
    Bluetoothprogramming
        for general info see protocols->bluetooth
        most communication between BLE devices happen through GATT server/client relationships, or through BLE mesh networks?
            You are correct in that most applications out there will be using the GATT server/clientrelationship. This is because when Bluetooth Low Energy was introduced in 2010(and later launched with iPhone 4s through CoreBluetooth in 2011), this was the only method of communication. Since then, subsequent releases of BLE introduced newer methods of communication:-
                *  LE L2CAP (introduced in BT v4.1, 2013) where lower level communication channels were used for fast and direct data transfer.
                *  LE Mesh (introduced in 2017) where most of the communication is based on BLE adverts and therefore any device that is on v4.0 can theoretically support it.
            The problems with both of these methods are the relative complexity and the slow adoption by vendors. As such, my recommendation is to continue using GATT examples/applications until you're more familiar with BLE and then proceed to using the other methods of communication.
        GAT
            https://docs.zephyrproject.org/latest/guides/bluetooth/bluetooth-arch.html
            defines four distinct roles
                Connection-oriented roles
                    Peripheral (e.g. a smart sensor, often with a limited user interface)
                    Central (typically a mobile phone or a PC)
                Connection-less roles
                    Broadcaster (sending out BLE advertisements, e.g. a smart beacon)
                    Observer (scanning for BLE advertisements)
                configuration option: CONFIG_BT_PERIPHERAL, CONFIG_BT_CENTRAL, CONFIG_BT_BROADCASTER & CONFIG_BT_OBSERVER
                 <https://docs.zephyrproject.org/latest/reference/kconfig/CONFIG_BT_PERIPHERAL.html#std-kconfig-CONFIG_BT_PERIPHERAL>
            Peripheral
                connectable advertising and expose one or more GATT services
                After registering servicesusing the bt_gatt_service_register()  API the application will typically start connectable advertising using the bt_le_adv_start()  API.
                 <https://docs.zephyrproject.org/latest/reference/bluetooth/gatt.html#c.bt_gatt_service_register>
                structure of server program
                (generates data)
                    nitialize the stack
                        bt_enable()
                    Register GATT service database
                        bt_gatt_register(services) - zie ik niet in peri_HR bv
                    Advertise and let others connect
                        bt_le_adv_start(parameters)
                    Notify of value changes
                        bt_gatt_notify(parameters)
                        Attribute value changes can be notified using bt_gatt_notify()
                         <https://docs.zephyrproject.org/latest/reference/bluetooth/gatt.html?highlight=bt_gatt_ccc#c.bt_gatt_notify>
                structure of client program
                    scan for devices
                    connect
                    discover its services and characteristics
                        server responds with available services+handles(=id of attribute)
                    read-request to server
                    Subscriptions to notification and indication can be initiated with use of bt_gatt_subscribe()
                     <https://docs.zephyrproject.org/latest/reference/bluetooth/gatt.html?highlight=bt_gatt_ccc#c.bt_gatt_subscribe>
                Attributes can be declaredusing the bt_gatt_attr struct or using one of the helper macros:
                 <https://docs.zephyrproject.org/latest/reference/bluetooth/gatt.html#c.bt_gatt_attr>
                Attribute value changes can be notified using bt_gatt_notify()  API, alternatively there is bt_gatt_notify_cb() where is is possible to pass a callback to be called when it is necessary to know the exact instant when the data has been transmitted over the air.
                 <https://docs.zephyrproject.org/latest/reference/bluetooth/gatt.html#c.bt_gatt_notify>
            Central
                To initially discover a device to connect to the application will likely use the bt_le_scan_start() API, wait for an appropriate device to be found (using the scan callback), stop scanning using bt_le_scan_stop()  and then connect to the device using bt_conn_create_le(). If the central wants to keep automatically reconnecting to the peripheral it should use the bt_le_set_auto_conn() API.
                 <https://docs.zephyrproject.org/latest/reference/bluetooth/gap.html#c.bt_le_scan_start>
        Change ADV interval
            populate a bt_le_adv_paraminstance using BT_LE_ADV_PARAMand pass this as the first argumentin the call to bt_le_adv_start())
        tools
            deprecated/not recommended
                Many tutorials on the internet are done with command-line tools that have been deprecated, such as hcitool and hcidump. If you see tutorials using the HCI (Host Controller Interface) socket then it is either out-of-date or at such a low level that it is best to stay away.
                 <https://en.wikipedia.org/wiki/List_of_Bluetooth_protocols#HCI>
                hcisocketbypasses the bluetoothd that is running on the Linux system that is used by the desktop tools. Using this is not a great idea unless you really, really know what you are doing.
            The command-line tools recommendedby the BlueZ developers are bluetoothctl  or, if you need more control, btmgmt. And instead of using hcidump, use btmon.
        PYTHON
            https://ukbaz.github.io/howto/bluetooth_overview.html
            list of Python libraries
                https://github.com/ukBaz/python-bluezero/wiki
                pyBluez- 220130 - not maintained anymore
            MGMT Socket
                The BlueZ Bluetooth Mamagement API is the next step up and the lowest level that the BlueZ developer recommend.
                 <https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/doc/mgmt-api.txt>
                The problem for Python users is this bug makes it difficult to access the mgmt socket. There are other duplicate bugs on this in the system. Until they are fixed, this remains off bounds for many Python users.
                 <https://bugs.python.org/issue36132>
            Bluez has DBus API
                python3
                >>> import pydbus
                >>> bus = pydbus.SystemBus()
                >>> adapter = bus.get('org.bluez', '/org/bluez/hci0')
                >>> dir(adapter)
                ['ActiveInstances', 'Address', 'AddressType', 'Alias', 'Class', 'ConnectDevice', 'Discoverable', 'DiscoverableTimeout', 'Discovering', 'Get', 'Ge
                >>> adapter.Name
                >>> adapter.Powered
                True
                >>> adapter.Address
                E8:2A:EA:AD:8B:2F'
            Classic
                RFCOMM (Or is that SPP?)
                    This is the most useful profile in classic mode for many activities in the maker community when you want ot exchange information between two boards that support Bluetooth serial connection. From Python 3.3 this is supported within the standard socket library.Below is an example of a client connecting to a server. This assumes the pairing has already happened and will do the connection.
                    >>> import socket
                    >>> s = socket.socket(socket.AF_BLUETOOTH, socket.SOCK_STREAM, socket.BTPROTO_RFCOMM)
                    >>> s.connect(('B8:27:EB:22:57:E0', 1))
                        NB: how do you know this BT address
                    >>> s.send(b'Hello')
                    >>> s.recv(1024)
                    b'world'
                    >>> s.close()
            BLE
                With BLE there is not the same level of support from native Python so it is required to use the DBus API. This means using the Device and GATT.
                 <https://git.kernel.org/pub/scm/bluetooth/bluez.git/tree/doc/device-api.txt>
                The difficult piece with these is that it is not known ahead of connection what the DBus Object Path will be for the devices, GATT Services, and GATT Characteristics we are interested in.
                This results in the need to do a reverse look-up from the UUID to the object path. This was the subject of a kata I held at my local Python user group.
                 <https://github.com/campug/bzero_kata>
            dit?
                So one dongle runs a standalone Zephyrapplication and the other runs hci_usb?
                https://github.com/Oxymoron79/zephyr-ble-peripheral
