Root File System
===

It's based on http://archlinuxarm.org/platforms/armv5/olinuxino

Modifications to the root file system
---

- New kernel (3.12.0)
- Serial tty for auart1 is enabled

  ```
  $: sudo mount /dev/mmcblk0p2 /mnt/olinuxino/
  $: sudo ln -s /usr/lib/systemd/system/serial-getty@.service /mnt/olinuxino/etc/systemd/system/getty.target.wants/serial-getty@ttyAPP1.service
  $: sudo umount /mnt/olinuxino
  ```

  ```
  DUART: early boot log
  AUART: 115200 8N1
  login: root
  Password: root
  ```

- Installed wireless tools: http://www.jann.cc/2013/02/04/2013/12/01/arch_linux_arm_network_tools_missing.html
- Installed updates: pacman -Syu, http://www.jann.cc/2013/02/04/a_new_image_for_the_imx233_olinuxino.html
- Installed tools: pacman -Sy usb_modeswitch wvdial base-devel vim screen git watchdog fswebcam python2 python2-pip i2c-tools

