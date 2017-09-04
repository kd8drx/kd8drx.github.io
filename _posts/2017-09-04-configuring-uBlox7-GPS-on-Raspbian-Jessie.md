---
layout: post
title: Configuring a U-Blox 7 GPS Receiver on Raspbian Jessie
tags: Ham_Radio Raspberry_Pi
---
One of my never-ending projects is building a packet radio "Go Kit", [largely based around Ed, W6ELA's "APRS Box" concept](http://elafargue.github.io/aprs-box). After some extensions to add Packet BBS, [Winlink](http://getpat.io/), and [AREDN](http://www.aredn.org) support, I've started calling the thing a "PiComm" unit. One day I'll write up more on it.

One of the more critical components of the setup is a GPS receiver: this provides location info for APRS, and allows the Raspberry Pi (which has no Real-Time Clock) to know what time it is without accessing the internet. I grabbed [this unit](https://www.amazon.com/gp/product/B01N01W8SK/ref=oh_aui_search_detailpage?ie=UTF8&psc=1), which had great reviews and 1-day shipping.

I am an impatient ham, after all.

Once it arrived, I installed GPSD on Raspbian Jesse and plugged it in. In theory, the system should have seen the GPS device appear on USB and automatically started GPSd. Except...it didn't. Worried I got a dead unit, I did some digging and found the device was present by running `lsusb -v` and `dmesg | grep -i usb`, and which showed the device mounting at `/dev/ttyACM0`. Running `cat /dev/ttyACM0` got me lots of raw GPS data, too -  the receiver was fine. So why wasn't GPSd starting automatically?

On Linux the job of starting services or auto-running commands when devices are plugged in are handled by a service called `uDev`, which uses a somewhat [cryptic language](http://www.reactivated.net/writing_udev_rules.html) to define rules that trigger actions - like auto-mounting a USB hard drive when it's plugged in. When installed on Raspbian, GPSd automatically defines rules for many receivers in `/lib/udev/rules.d/60-gpsd.rules` - but doesn't include rules for u-Blox 7 receivers:

```
# u-blox AG, u-blox 5 (tested with Navilock NL-402U) [linux module: cdc_acm]
ATTRS{idVendor}=="1546", ATTRS{idProduct}=="01a5", SYMLINK+="gps%n", TAG+="systemd", ENV{SYSTEMD_WANTS}="gpsdctl@%k.service"
# u-blox AG, u-blox 6 (tested with GNSS Evaluation Kit TCXO) [linux module: cdc_acm]
ATTRS{idVendor}=="1546", ATTRS{idProduct}=="01a6", SYMLINK+="gps%n", TAG+="systemd", ENV{SYSTEMD_WANTS}="gpsdctl@%k.service"
```

Admittedly, that looks _pretty close_ to what I saw earlier from running `lsusb -v` - but not close enough:
```
Bus 001 Device 008: ID 1546:01a7 U-Blox AG
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               1.10
  bDeviceClass            2 Communications
  bDeviceSubClass         0
  bDeviceProtocol         0
  bMaxPacketSize0        64
  idVendor           0x1546 U-Blox AG
  idProduct          0x01a7
  bcdDevice            1.00
  iManufacturer           1
  iProduct                2
  iSerial                 0
  bNumConfigurations      1
```

Everything is the same with the uBlox 7 chipset, except the `idProduct` is different: 01a7, vs. 01a6 or 01a5. So, to get uDev to launch GPSd automatically when I plug it in (or when the system boots), I just copy one of the existing lines and edit the `idProduct` to match mine:

```
# u-blox AG, u-blox 7 (Tested with VANWEI VK-162)  [linux module: cdc_acm]
ATTRS{idVendor}=="1546", ATTRS{idProduct}=="01a7", SYMLINK+="gps%n", TAG+="systemd", ENV{SYSTEMD_WANTS}="gpsdctl@%k.service"
```

Reload the uDev rules and et voilà! Even better, GPSd re-maps the devices entry in /dev to `/dev/gps0`, which makes it easy to configure other apps like Polaric to grab position data - regardless of what receiver you use.
