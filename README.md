
Linux for the Bones iMX233 Audio Development Board
===

Please check out the following links for more information:

- http://blog.bones-embedded.ch/imx233-audio-development-board/
- http://www.jann.cc/2013/08/31/porting_linux_to_a_new_board.html
- http://www.jann.cc/2012/08/23/building_a_kernel_3_x_for_the_olinuxino_from_sources.html
- https://github.com/koliqi/imx23-olinuxino

You can download a complete SD card image from here:
http://sourceforge.net/projects/janncc/files/imx233-audio-development-board/entire-sdcard/2013-12-01/

Windows quick start guide
---

* You will need an SD card of at least 2GB
* Download the Win32DiskImager http://sourceforge.net/projects/win32diskimager/
* Download a complete SD card image from above
* Use Win32DiskImager to write the image to the SD card, insert it into your board
  and power it up

Building a kernel 3.12 for the Bones iMX233 Audio Development Board
===

Software requirements
---

You will need an ARM bare metal cross-compiler. Its recommended to use the compiler 
from Launchpad, at least if you encounter any problems:
- https://launchpad.net/gcc-arm-embedded/+download


Hardware requirements
---

SD card with two partitions:
- boot partition for kernel, and 
- [rootfs](Root-Filesystem.md) partition with your preferred Linux distribution.

Bones iMX233 Audio Development Board or compatible hardware:
- http://blog.bones-embedded.ch/imx233-audio-development-board/

Getting the code
===
```
$: git clone https://github.com/christianjann/imx233-audio-development-board.git
```

This repository contains also:
* Freescale imx-bootlets and the utility elftosb2,
* patches for imx-bootlets and the kernel.

Freescale imx-bootlets and the utility elftosb2 are from the Freescale package L2.6.31_10.05.02_ER_source.
The utility elftosb2 is distributed in binary form. Sources can be found at the following link:
http://repository.timesys.com/buildsources/e/elftosb/elftosb-10.12.01/


1) Building a kernel from sources
===

Switch to the directory kernel and download the kernel sources:
```
$: cd imx233-audio-development-board/kernel
$: wget http://www.kernel.org/pub/linux/kernel/v3.0/linux-3.12.tar.bz2
$: tar xvjf linux-3.12.tar.bz2
$: mv linux-3.12 linux-stable
```

The new directory linux-stable contains the kernel sources. Switch to the directory linux-stable to apply patches.
```
$: cd linux-stable
$: patch -p1 < ../0001-ARM-bones-badb-Add-auart1.patch
$: patch -p1 < ../0002-ARM-imx23-olinuxino-Add-i2c-support.patch
$: patch -p1 < ../0003-ARM-bones-badb-Enable-chip-detect-and-use-pull-up.patch
$: patch -p1 < ../0004-ARM-bones-badb-Add-the-main-SD-card.patch
$: patch -p1 < ../0005-ARM-bones-badb-Make-the-status-led-work.patch
$: patch -p1 < ../0006-ARM-bones-badb-Removed-duplicated-ssp1.patch
```

Or checkout the development snapshot:
```
$: git clone -b badb-3.12 https://github.com/christianjann/linux-badb linux-stable
```

Configure the kernel
---
Start from the supplied default mxs_defconfig configuration:
```
$: make ARCH=arm CROSS_COMPILE=arm-none-eabi- mxs_defconfig
$: cp ../dotconfig .config # maybe use my premade kernel configuration
$: make ARCH=arm CROSS_COMPILE=arm-none-eabi- menuconfig
```
Select `Boot options --->` and select the following options:
```
 Boot options ---->
  [*]  Use appended device tree blob to zImage (EXPERIMENTAL)
  [*]   Supplement the appended DTB with traditional ATAG information
          Kernel command line type (Use bootloader kernel arguments if available)  --->
```

Enable the application UART:
```
Device Drivers  --->
  Character devices  --->
    Serial drivers  --->
      <*> MXS AUART support
      [*]   MXS AUART console support
```

Enable CGROUPS, AUTOFS4, FANOTIFY and DEVTMPFS (recommended for systemd):
```
General setup  --->
  [*] Control Group support  --->
File systems  --->
  [*] Filesystem wide access notification
  <*> Kernel automounter version 4 support (also supports v3)
```

Enable dynamic printk() support (CONFIG_DYNAMIC_DEBUG):
```
Kernel hacking  --->
  printk and dmesg options  --->
    [*] Enable dynamic printk() support
```

Enable the driver for USB serial adapters:
```
Device Drivers  --->
  [*] USB support  --->
    <*>   USB Serial Converter support  --->
      [*]   USB Generic Serial Driver
```

Drivers for most USB WiFi adapters and some other drivers are enabled too, see [here](http://www.jann.cc/2012/08/23/building_a_kernel_3_x_for_the_olinuxino_from_sources.html).

Activate other drivers as needed. Save and exit from the menuconfig application.

Compile the kernel
---
```
$: make -j4 ARCH=arm CROSS_COMPILE=arm-none-eabi- zImage modules
```
The kernel is ready at arch/arm/boot/zImage

Create the device tree blob .dtb file:
```
$: make ARCH=arm CROSS_COMPILE=arm-none-eabi- imx23-olinuxino.dtb
```
Join zImage and imx23-olinuxino.dtb into a new file zImage_dtb:
```
$: cat arch/arm/boot/zImage arch/arm/boot/dts/imx23-olinuxino.dtb > arch/arm/boot/zImage_dtb
```
If you want to repeat this procedure, start with clean-up:
```
$: make ARCH=arm CROSS_COMPILE=arm-none-eabi- distclean
$: make ARCH=arm CROSS_COMPILE=arm-none-eabi- clean
```

2) Bootlets
===

The iMX23 SoC contains a built-in ROM firmware capable of loading and
executing binary images in special format from different locations including
MMC/SD card and NAND flash. Binary image is called a boot stream (.bs).
Boot stream consists of a series of smaller bootable images (bootlets)
such clock bootlet, power bootlet etc.
Linking bootlets with kernel and converting from elf format to raw boot stream is
done with utility elftobs.
In this package, utility elftobs2 is located in directory elftosb-0.3.

Switch to the directory elftosb-0.3 and create a symbolic link into the compilers default
PATH:
```
$: sudo ln -s `pwd`/elftosb2 /usr/sbin/
```
Check with locate:
```
$: locate elftosb2
```
elftosb2 should be located at /usr/sbin/elftosb2.
 
Next, switch to the directory boot and untar the archive imx-bootlets-src-10.05.02.tar.gz
```
$: tar xvzf imx-bootlets-src-10.05.02.tar.gz
```
then switch to the directory imx-bootlets-src-10.05.02 and apply the patches:
```
$: patch -p1 < ../imx23_olinuxino_bootlets.patch
$: patch -p1 < ../imx23_babd_bootlets.patch
```
This patched package requires zImage in this directory. We have created
zImage_dtb instead, so create a symbolic link:
```
$: ln -s ../../kernel/linux-stable/arch/arm/boot/zImage_dtb ./zImage

```

Create the boot stream file:
```
$: make CROSS_COMPILE=arm-none-eabi- clean
$: make CROSS_COMPILE=arm-none-eabi-
```
The final response should be:
```
To install bootstream onto SD/MMC card, type: sudo dd if=sd_mmc_bootstream.raw of=/dev/sdXY
where X is the correct letter for your sd or mmc device (to check, do a ls /dev/sd*) and Y is 
the partition number for the bootstream.
```
In my system the sd device is /dev/mmcblk0p1, so writing bootstream file to the SD card is done by:
```
$: sudo dd if=sd_mmc_bootstream.raw of=/dev/mmcblk0p1
```
Card is ready.

Partition layout:
```
[++-------------------------/dev/mmcblk0-----------------------------------]
###[+/dev/mmcblk0p1-][+------/dev/mmcblk0p2---------][+---/dev/mmcblk0p3---]
###[+-----kernel----][+----------rootfs-------------][+-------swap---------]

###, +: partition table and other file system information
-: free space
```

It is good practice to work with multiple consoles. Open one into directory linux-mainline,
second into imx-bootlets-src-10.05.02 and third console for minicom.


