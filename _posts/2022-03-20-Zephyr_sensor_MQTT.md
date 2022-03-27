---
layout: post
title: Zephyr sensors and MQTT - Arduino nano 33 BLE sense
---
This posts shows how the sensors on the Arduino nano 33 BLE sense can be used in Zephyr.

All files needed to build this program can be found at `https://github.com/basaandewiel/zephyr_hts221_mqtt`. 

I tried to use sensors of my arduino, but found out that the sensors are not defined in the standard dts (device tree) file supplied for this board (in directory ...zephyrprojects/zephyr/boards/arm/).
After some searching I found this site https://devzone.nordicsemi.com/f/nordic-q-a/77845/problem-initializing-i2c-on-arduino-nano-33-ble.

It does not only provide an updated device tree with sensors, but also solved a bug in the initialization of the sensors.
The following device tree must be placed into `~/zephyrproject/zephyr/boards/arm/arduino_nano_33_ble`

```
/*
 * Copyright (c) 2020 Jefferson Lee
 *
 * SPDX-License-Identifier: Apache-2.0
 */
/dts-v1/;
#include <nordic/nrf52840_qiaa.dtsi>

/ {
        model = "Arduino Nano 33 BLE (Sense)";
        compatible = "arduino,arduino_nano_33_ble";

        chosen {
                zephyr,console = &uart0;
                zephyr,shell-uart = &uart0;
                zephyr,uart-mcumgr = &uart0;
                zephyr,bt-mon-uart = &uart0;
                zephyr,bt-c2h-uart = &uart0;
                zephyr,sram = &sram0;
                zephyr,flash = &flash0;
                zephyr,code-partition = &code_partition;
        };

        leds {
                compatible = "gpio-leds";
                led0: led_0 {
                        gpios = <&gpio0 24 0>;
                        label = "Red LED";
                };
                led1: led_1 {
                        gpios = <&gpio0 16 0>;
                        label = "Green LED";
                };
                led2: led_2 {
                        gpios = <&gpio0 6 0>;
                        label = "Blue LED";
                };
        };

        /* These aliases are provided for compatibility with samples */
        aliases {
                led0 = &led0;
                                                                 1,1           Top
```

Following is also copied from the link mentioned above:
Additionally, I changed zephyr/boards/arm/arduino_nano_33_ble/src/init_sensors.c such that initializing function is called at APPLICATION instead of PRE_KERNEL_1. Not sure why but initializing GPIO devices will cause the device to go fault at Pre_KERNEL_1.
```
/*
 * Copyright (c) 2020 Jefferson Lee.
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <init.h>
#include <arduino_nano_33_ble.h>

/*
 * this method roughly follows the steps here:
 * https://github.com/arduino/ArduinoCore-nRF528x-mbedos/blob/6216632cc70271619ad43547c804dabb4afa4a00/variants/ARDUINO_NANO33BLE/variant.cpp#L136
 */

static int board_internal_sensors_init(const struct device *dev)
{
	ARG_UNUSED(dev);
	struct arduino_gpio_t gpios;

	arduino_gpio_init(&gpios);

	arduino_gpio_pinMode(&gpios, ARDUINO_LEDPWR, GPIO_OUTPUT);
	arduino_gpio_digitalWrite(&gpios, ARDUINO_LEDPWR, 1);

	CoreDebug->DEMCR = 0;
	NRF_CLOCK->TRACECONFIG = 0;

	/*
	 * Arduino uses software to disable RTC1,
	 * but I disabled it using DeviceTree
	 */
	/* nrf_rtc_event_disable(NRF_RTC1, NRF_RTC_INT_COMPARE0_MASK); */
	/* nrf_rtc_int_disable(NRF_RTC1, NRF_RTC_INT_COMPARE0_MASK); */

	NRF_PWM_Type * PWM[] = {
		NRF_PWM0, NRF_PWM1, NRF_PWM2, NRF_PWM3
	};

	for (unsigned int i = 0; i < (ARRAY_SIZE(PWM)); i++) {
		PWM[i]->ENABLE = 0;
		PWM[i]->PSEL.OUT[0] = 0xFFFFFFFFUL;
	}

	/*
	 * the PCB designers decided to use GPIO's
	 * as power pins for the internal sensors
	 */
	arduino_gpio_pinMode(&gpios, ARDUINO_INTERNAL_VDD_ENV_ENABLE, GPIO_OUTPUT);
	arduino_gpio_pinMode(&gpios, ARDUINO_INTERNAL_I2C_PULLUP, GPIO_OUTPUT);
	arduino_gpio_digitalWrite(&gpios, ARDUINO_INTERNAL_VDD_ENV_ENABLE, 1);
	arduino_gpio_digitalWrite(&gpios, ARDUINO_INTERNAL_I2C_PULLUP, 1);
	return 0;
}
SYS_INIT(board_internal_sensors_init, APPLICATION, 32);
```

First of all I wanted to test whether the sensors are working; so I initialised the temperature and humidity sensors and printed their values to the console.
The following program is working, but *during the first boot the sensors are sometimes not recognized; after a reboot it is always working; I do not yet know why the behaviour after the first boot is not correct.*

```
/*
 * Copyright (c) 2017 Linaro Limited
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <zephyr.h>
#include <device.h>
#include <drivers/sensor.h>
#include <stdio.h>
#include <sys/util.h>

#include <sys/printk.h>
#include <usb/usb_device.h>
#include <drivers/uart.h>

const struct device *dev;

static void process_sample(const struct device *dev)
{
	struct sensor_value temp, hum;
	if (sensor_sample_fetch(dev) < 0) {
		printk("Sensor sample update error\n");
		return;
	}

	if (sensor_channel_get(dev, SENSOR_CHAN_AMBIENT_TEMP, &temp) < 0) {
		printk("Cannot read HTS221 temperature channel\n");
		return;
	}

	if (sensor_channel_get(dev, SENSOR_CHAN_HUMIDITY, &hum) < 0) {
		printk("Cannot read HTS221 humidity channel\n");
		return;
	}

	/* display temperature */
	printk("Temperature:%.1f C\n", sensor_value_to_double(&temp));

	/* display humidity */
	printk("Relative Humidity:%.1f%%\n",
	       sensor_value_to_double(&hum));
}

void init_usb_serial(void) {
	uint32_t dtr = 0;

	if (usb_enable(NULL)) {
		return;
	}

	/* Poll if the DTR flag was set */
	dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_console));
	while (!dtr) {
		uart_line_ctrl_get(dev, UART_LINE_CTRL_DTR, &dtr);
		/* Give CPU resources to low priority threads. */
		k_sleep(K_MSEC(100));
	}
	k_sleep(K_MSEC(5000)); //give user time to start 'screen /dev/ttyACM0'
}

void main(void)
{
	init_usb_serial();	//so we can printk to console

    dev = device_get_binding("HTS221");  //string must match string in device tree
 	//dev = device_get_binding(DT_LABEL(DT_INST(0, st_hts221)));

	if (dev == NULL) {
		printk("Could not get HTS221 device\n");
		return;
	}

	while (true) {
		process_sample(dev);
		k_sleep(K_MSEC(2000));
	}
}
```





