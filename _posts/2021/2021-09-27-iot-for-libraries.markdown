---
layout: post
title: IoT for libraries
date: '2021-09-27 10:15'
excerpt: IoT for libraries
comments: false
---

IoT for libraries
=================


**To measure or not to measure that's the question.**

Overview
--------

As the cost of sensors are dropping dramatically (Despite recent COVID hickups) libraries should invest some time and general understanding of howto deploy these. The IoT landscape is riddled with commercial companies that want to gather as much data as they can, while I think that a library thrives on privacy. My advise for any library that want's to deploy a massive IoT-network, please be transparent about it, involve your patrons, they have a right to know and might be interested in the subject as well. Since I am a hand's on type of person I want to share some experiments I did, and talk about cost effectiveness of these kind of devices and howto wire them up. I did all experiments during the COVID-lockdown period, so no book-lovers where harmed in the process.

<img src="https://s3.eu-central-1.amazonaws.com/centaur-wp/econsultancy/prod/content/uploads/archive/images/resized/0008/6869/atlas_bjsmcfal_2x-blog-flyer.png" alt="Cost of sensors">

A good article about other experiments done inside an actual library appeared in code4lib Issue 38, 2017-10-18.

[Code4Lib Testing Three Types of Raspberry Pi People Counters](https://journal.code4lib.org/articles/12947)

Let's get our definition straight on what IoT devices are. I'm a fan of Wikipedia so here is the obligatory quote:

> The Internet of Things (IoT) describes physical objects (or groups of such objects), that are embedded with sensors, processing ability, software, and other technologies, and that connect and exchange data with other devices and systems over the Internet or other communications networks.

There are a lot of good resources on the net about how-to setup your own IoT landscape. My weapon of choice for setting thins up is Python. There is a Python distribution available especially for playing with these devices, MicroPython. I highly recommend this book:

- Programming with MicroPython,
- Embedded programming with microcontrollers & Python
- Nicholas H. Tollervey
- ISBN: 978-1-491-97273-1

Now that we've defined our programming language let's talk hardware, how should you setup your IoT landscape.
The general idea here is this:

Sensor -> Microcontroller -> Raspberry Pi

Sensors
-------
Let's start with the sensor part. There are a lot of things you can measure with sensors, ranging from simple switches to water-levels, radar sensors, CO2 sensors ect.

To explore a wide range of possibilities I suggest getting a sensor-kit, something like this:
![Sensor kit example](https://raw.githubusercontent.com/WillemJan/willemjan.github.io/master/_posts/2021/sensor-kit.jpg)

Most of these sensor have been tested with MicroPython and tutorials on howto connect and operate these are widely available, as well as source code.

Microcontrollers
----------------
I've tested several devices for this purpose, and the thing I like best and is super-cheap to deploy, the ESP8266 aka as NodeMCU.
For all expiriments I used Debian 10.10 (Buster) on my laptop and python3 to communicate with the ESP8266 microcontroller, but this can also be done from the Raspberry Pi.

<img src="https://raw.githubusercontent.com/WillemJan/willemjan.github.io/master/_posts/2021/esp8266.png" align="right" alt="ESP8266 controller">

According to Wikipedia: 

> The ESP8266 is a low-cost Wi-Fi microchip, with a full TCP/IP stack and microcontroller capability, produced by Espressif Systems in Shanghai, China. 

At the time of writing this blog, getting one module will cost you (In the Netherlands):

```
  Cost of an ESP8266: € 2,53
  Cost of shipping:   € 1,44 +
                        =
  Total cost:         € 3,97
```

There are several ways of communicating with the chip, once deployed.

- Option 1) Via serial communication using the USB-connection.
- Option 2) Via WiFi.
- Option 3) By other means, like the SPI bus or using a cheap GMS module.

I've explored Option 1 and Option 2 in depth, and will share my experience here.

The first step is to erase and flash new firmware onto the ESP8266 device.

First install the required tools and firmware:
```
sudo apt install -y picocom esptool 
pip3 install adafruit-ampy
mkdir esp_8266
touch boot.py # For now make an empty boot.py, later you can fill this with code to read and transmit sensor data.
wget 'http://micropython.org/resources/firmware/esp8266-20210902-v1.17.bin'
curl -s https://raw.githubusercontent.com/micropython/micropython-lib/master/micropython/urllib.urequest/urllib/urequest.py > ureq.py
```

Note that esptool may be outdated, if you get wierd errors during invocation, use 'sudo apt remove -y esptool; sudo pip3 install esptool'
More info on the [firmware](http://micropython.org/download/esp8266/).

This is my little flash & disaster recovery script:
```
#!/usr/bin/env bash

esptool.py --port /dev/ttyUSB0 erase_flash
esptool.py --port /dev/ttyUSB0 --baud 115200 write_flash --flash_size=detect 0 /home/aloha/fe2/external/bin/esp8266-20190529-v1.11.bin 

ampy  -p /dev/ttyUSB0 put boot.py
ampy  -p /dev/ttyUSB0 ls
ampy  -p /dev/ttyUSB0 mkdir urequests
ampy  -p /dev/ttyUSB0 put ureq.py /urequests/__init__.py
ampy  -p /dev/ttyUSB0 get boot.py

picocom --baud 115200 /dev/ttyUSB0
```

Using the serial connection you will be able te transfer data very reliable, but not as fast as over WiFi (2.7 mega bits/sec) according to [load tesing an esp8266](https://arunoda.me/blog/load-testing-an-esp8266).
But for low-latency and high reliability/security stuff a serial connection works just fine, I've tested the Python library 'pyserial' to get reading directly from the USB-port and this works just fine.

Installing pyserial:
```
sudo pip3 install pyserial
```

Code for serial communication with the ESP8266:
```
#!/usr/bin/env python3

import serial

ser = serial.Serial('/dev/ttyUSB0', 115200)
while True:
    print(ser.readline().decode("utf-8").strip())
```

Please note the port, which by default will be '/dev/ttyUSB0' under Debian, it might be different on your OS, if unsure check the output of 'sudo dmesg'

By default the ESP8266 turns on a Wifi-AP, if you use a serial connection it's wise to turn this off completely using the following code:
```
import network
sta_if = network.WLAN(network.STA_IF)
sta_if.active(False)
ap_if = network.WLAN(network.AP_IF)
ap_if.active(False)
```

I prefer to let the ESP8266 send data, rather then having the Raspberry Pi poll all the ESP8266's deployed, so I recommend turning off the access point (Which will by default show up something like 'MicroPython-2884894' in your WiFi-network list).
In order to do this, the last 2 lines of the code-snippet above will have to run first, before starting the main loop, add them to the boot.py file.

Raspberry Pi
------------
The Raspberry Pi acts as the IoT-gateway in this setup, and can be used to power the microcontrollers via USB Cable A Male to Micro B Female.
