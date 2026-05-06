---
title: "Cross compilation of a Kernel module on Ubuntu for Raspbian"
date: 2024-01-16
categories: 
  - "it"
image: "cross-compilation-raspberry-pi.png"
description: "A complete tutorial on cross-compiling kernel modules for Raspbian using an Ubuntu machine. Discover the workflow to retrieve firmware hashes, prepare the kernel source, and compile a custom Realtek WiFi driver for your Raspberry Pi."
---

In order to do a cross compilation of a Kernel module on Ubuntu for Raspbian, we'll take as an example the D-Link DWA-131 WiFi dongle that is not supported out of the box on [Raspbian](https://www.raspberrypi.com/software/), a compilation is required. You have two options :

- Compile on the Pi

- Cross-compile on a other machine.

I prefer to cross-compile as it’s faster and easier : you avoid to install the compilation tools and download huge sources on a small SD card…

## Step 1 : Collect information

### Dongle information

```
>lsusb
Bus 001 Device 002: ID 0424:9514 Standard Microsystems Corp.
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp.
Bus 001 Device 004: ID 2001:3319 D-Link Corp.
```

### Information needed

Get the firmware hash:

```
FIRMWARE_HASH=$(zgrep "* firmware as of" /usr/share/doc/raspberrypi-bootloader/changelog.Debian.gz | head -1 | awk '{ print $5 }')
```

Print it :

```
echo $FIRMWARE_HASH
```

Result :

```
3442862c10fab68c2e88d660d2e69c143bb1f00c
```

Get the kernel hash:

```
KERNEL_HASH=$(wget https://raw.github.com/raspberrypi/firmware/$FIRMWARE_HASH/extra/git_hash -O -)
```

Print it :

```
echo $KERNEL_HASH
```

Result :

```
ca312f557513e057c456598528e663fe9d009498
```

We need to load the configs module to be able to retrieve the configuration later on:

```
sudo modprobe configs
```

And finally your Kernel version :

```
uname -r
```

Result :

```
4.1.17+
```

Copy paste these information somewhere in a text file, we’ll use it later on.

## Step 2 : On the compilation server

### Preparation

Prepare a folder to work in :

```
mkdir ~/raspberrypi
export BASEDIR=~/raspberrypi
cd $BASEDIR
```

Create variables based on what we found earlier :

```
FIRMWARE_HASH=3442862c10fab68c2e88d660d2e69c143bb1f00c
KERNEL_HASH=ca312f557513e057c456598528e663fe9d009498
```

### Kernel source

Download and extract the kernel source :

```
wget https://github.com/raspberrypi/linux/archive/$KERNEL_HASH.tar.gz
tar -xzf $KERNEL_HASH.tar.gz
export KERNEL_SRC=$BASEDIR/linux-$KERNEL_HASH
```

### Kernel configuration

Download the configuration from the raspberry pi. The IP should be the IP of the raspberry pi :

```
scp pi@192.168.137.250:/proc/config.gz .
```

### Compilation tools

Download the compilation tools :

```
cd $BASEDIR
git clone git://github.com/raspberrypi/tools.git
export CCDIR=$BASEDIR/tools/arm-bcm2708/arm-bcm2708-linux-gnueabi/bin
export CCPREFIX=$CCDIR/arm-bcm2708-linux-gnueabi-
```

### Kernel preparation

Prepare the Kernel source :

```
cd $KERNEL_SRC
zcat $BASEDIR/config.gz > .config
wget https://raw.githubusercontent.com/raspberrypi/firmware/$FIRMWARE_HASH/extra/Module.symvers
ARCH=arm CROSS_COMPILE=${CCPREFIX} make oldconfig
ARCH=arm CROSS_COMPILE=${CCPREFIX} make prepare
```

### Driver source

Download the driver source :

```
cd $BASEDIR
git clone https://github.com/romcyncynatus/rtl8192eu.git
cd rtl8192eu/
```

### Driver compilation

Compilation :

```
make ARCH=arm CROSS_COMPILE=$CCPREFIX KVER=4.1.17+ KSRC=$KERNEL_SRC 
```

First error :

```
make[1]: Entering directory '/home/ron/raspberrypi/linux-ca312f557513e057c456598528e663fe9d009498'
  CC [M]  /home/ron/raspberrypi/rtl8192eu/core/rtw_cmd.o
cc1: error: -Werror=date-time: no option -Wdate-time
scripts/Makefile.build:258: recipe for target '/home/ron/raspberrypi/rtl8192eu/core/rtw_cmd.o' failed
make[2]: *** [/home/ron/raspberrypi/rtl8192eu/core/rtw_cmd.o] Error 1
Makefile:1384: recipe for target '_module_/home/ron/raspberrypi/rtl8192eu' failed
make[1]: *** [_module_/home/ron/raspberrypi/rtl8192eu] Error 2
make[1]: Leaving directory '/home/ron/raspberrypi/linux-ca312f557513e057c456598528e663fe9d009498'
Makefile:1455: recipe for target 'modules' failed
make: *** [modules] Error 2
```

Edit the Makefile :

```
vi $BASEDIR/rtl8192eu/Makefile
```

Change the following line :

```
EXTRA_CFLAGS += -Wno-error=date-time
```

By :

```
#EXTRA_CFLAGS += -Wno-error=date-time
```

Second error :

```
  CC [M]  /home/ron/raspberrypi/rtl8192eu/os_dep/linux/rtw_android.o
/home/ron/raspberrypi/rtl8192eu/os_dep/linux/rtw_android.c: In function 'rtw_android_cmdstr_to_num':
/home/ron/raspberrypi/rtl8192eu/os_dep/linux/rtw_android.c:345:3: error: implicit declaration of function 'strnicmp' [-Werror=implicit-function-declaration]
cc1: some warnings being treated as errors
scripts/Makefile.build:258: recipe for target '/home/ron/raspberrypi/rtl8192eu/os_dep/linux/rtw_android.o' failed
make[2]: *** [/home/ron/raspberrypi/rtl8192eu/os_dep/linux/rtw_android.o] Error 1
Makefile:1384: recipe for target '_module_/home/ron/raspberrypi/rtl8192eu' failed
make[1]: *** [_module_/home/ron/raspberrypi/rtl8192eu] Error 2
make[1]: Leaving directory '/home/ron/raspberrypi/linux-ca312f557513e057c456598528e663fe9d009498'
Makefile:1455: recipe for target 'modules' failed
make: *** [modules] Error 2
```

Edit the rtw\_android.c:

```
vi $BASEDIR/rtl8192eu/os_dep/linux/rtw_android.c
```

Change the following line :

```
if(0 == strnicmp(cmdstr , android_wifi_cmd_str[cmd_num], strlen(android_wifi_cmd_str[cmd_num])) )
```

By :

```
if(0 == strncasecmp(cmdstr , android_wifi_cmd_str[cmd_num], strlen(android_wifi_cmd_str[cmd_num])) )
```

The compilation should be fine now !

### Driver transfer

We’ll use SCP to copy the module (8192eu.ko) on our Raspberry. The IP used here is the Raspberry Pi’s.

```
scp 8192eu.ko pi@192.168.137.250:~
```

## Step 3 : On the Raspberry Pi

After the transfer, the compiled driver should be in your home folder.

Activation :

```
sudo bash -c 'echo "options 8192eu rtw_power_mgnt=0 rtw_enusbss=0">/etc/modprobe.d/8192eu.conf'
sudo install -p -m 644 8192eu.ko /lib/modules/4.1.17+/kernel/drivers/net/wireless
sudo depmod 4.1.17+
```

Reboot:

```
sudo reboot
```

After a reboot, the module is loaded

```
>modinfo 8192eu
filename:       /lib/modules/4.1.17+/kernel/drivers/net/wireless/8192eu.ko
version:        v4.3.8_12406.20140929
author:         Realtek Semiconductor Corp.
description:    Realtek Wireless Lan Driver
license:        GPL
srcversion:     AFD4F17337FAE61BE3C568D
alias:          usb:v2001p3319d*dc*dsc*dp*ic*isc*ip*in*
alias:          usb:v0BDAp818Cd*dc*dsc*dp*icFFiscFFipFFin*
alias:          usb:v0BDAp818Bd*dc*dsc*dp*icFFiscFFipFFin*
```

And WiFi available :

```
wlan0     Link encap:Ethernet  HWaddr XX:XX:XX:XX:XX:XX
          inet addr:192.168.1.25  Bcast:192.168.1.255  Mask:255.255.255.0
          inet6 addr: fe80::5326:231f:6f24:cb1c/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:11598 errors:0 dropped:9 overruns:0 frame:0
          TX packets:209 errors:0 dropped:1 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:1841333 (1.7 MiB)  TX bytes:26125 (25.5 KiB)
```

You're done with cross compilation of a Kernel module on Ubuntu for Raspbian

