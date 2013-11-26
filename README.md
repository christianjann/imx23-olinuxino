
Linux for the Bones iMX233 Audio Development Board
===

Please check out the following links for more information:

- http://blog.bones-embedded.ch/imx233-audio-development-board/
- http://www.jann.cc/2013/08/31/porting_linux_to_a_new_board.html
- http://www.jann.cc/2012/08/23/building_a_kernel_3_x_for_the_olinuxino_from_sources.html
- https://github.com/koliqi/imx23-olinuxino

You can download a complete sdcard image from here:
http://sourceforge.net/projects/janncc/files/imx233-audio-development-board/entire-sdcard/2013-11-20/

Windows Quick start guide
---

* You need a SD card with at least 2 GB
* Download the Win32DiskImager http://sourceforge.net/projects/win32diskimager/
* Download a complete sdcard image from above
* Use Win32DiskImager to write the image to the SD card, insert it into your board
  and power it up

Building a kernel 3.12 for the Bones iMX233 Audio Development Board
===

Software requirements
---

Cross compiler and git.


Hardware requirements
---

SD card with two partition:
- boot partition for kernel, and 
- rootfs partition with your preferred Linux distribution (http://archlinuxarm.org/platforms/armv5/olinuxino).

Getting the code
===
```
$: git clone https://github.com/christianjann/imx233-audio-development-board.git
```

This package contain also:
* Freescale imx-bootlets and utility elftosb2,
* patches for imx-bootlets and the kernel.

Freescale imx-bootlets and utility elftosb2 are from Freescale package L2.6.31_10.05.02_ER_source.
Utility elftosb2 is distributed in binary form. Sources can be found at the following link:
http://repository.timesys.com/buildsources/e/elftosb/elftosb-10.12.01/


1) Building a kernel from sources
===

Switch into directory kernel and download kernel sources:
```
$: cd imx233-audio-development-board/kernel
$: wget http://www.kernel.org/pub/linux/kernel/v3.0/linux-3.12.tar.bz2
$: tar xvjf linux-3.12.tar.bz2
$: mv linux-3.12 linux-stable
```
New directory linux-stable contains kernel sources. Switch into directory linux-stable to apply patches.
```
$: cd linux-stable
$: patch -p1 < ../0001-ARM-imx23-olinuxino-Add-auart1-for-badb.patch
```

Configure kernel
---
Start from supplied default mxs_defconfig configuration:  
```
$: make ARCH=arm CROSS_COMPILE=arm-none-eabi- mxs_defconfig
$: make ARCH=arm CROSS_COMPILE=arm-none-eabi- menuconfig
```
Select `Boot options --->` and select following options:
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

Activate other drivers as needed. Save and exit from menuconfig application.

Compile kernel
---
```
$: make -j4 ARCH=arm CROSS_COMPILE=arm-none-eabi- zImage modules
```
Kernel is ready at arch/arm/boot/zImage

Create device tree blob .dtb file:
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

Switch into directory elftosb-0.3 and make symbolic link into compilers default
PATH:
```
$: sudo ln -s `pwd`/elftosb2 /usr/sbin/  
```
Check with  locate:
```
$: locate elftosb2
```
elftosb2 should be located at /usr/sbin/elftosb2.
 
Next, switch into directory boot and untar archive imx-bootlets-src-10.05.02.tar.gz
```
$: tar xvzf imx-bootlets-src-10.05.02.tar.gz
```
then switch into directory imx-bootlets-src-10.05.02 and apply patches:
```
$: patch -p1 < ../imx23_olinuxino_bootlets.patch
$: patch -p1 < ../imx23_babd_bootlets.patch
```
This patched package require zImage in this directory. We have created
zImage_dtb instead, so make symbolic link as:
```
$: ln -s ../../kernel/linux-stable/arch/arm/boot/zImage_dtb ./zImage

```

Make boot stream file:
```
$: make CROSS_COMPILE=arm-none-eabi- clean
$: make CROSS_COMPILE=arm-none-eabi-
```
Final response would be:
```
To install bootstream onto SD/MMC card, type: sudo dd if=sd_mmc_bootstream.raw of=/dev/sdXY
where X is the correct letter for your sd or mmc device (to check, do a ls /dev/sd*) and Y is 
the partition number for the bootstream.
```
In my system sd device is /dev/mmcblk0p1, so writing bootstream file in card is done by:
```
$: sudo dd if=sd_mmc_bootstream.raw of=/dev/mmcblk0p1
```
Card is ready.

Partition layout:
```
[++-------------------------/dev/mmcblk0-----------------------------------]
###[+/dev/mmcblk0p1-][+------/dev/mmcblk0p2---------][+---/dev/mmcblk0p3---]
###[+-----kernel----][+----------rootfs-------------][+-------swap---------]

###, +: partition table and other filesystem information
-: free space
```

It is good practice to work with multiple consoles. Open one into directory linux-mainline,
second into imx-bootlets-src-10.05.02 and third console for minicom to monitor olinuxino.


