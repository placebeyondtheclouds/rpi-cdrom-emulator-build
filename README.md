# DIY cdrom emulator build

## Description

Building a hardware cdrom emulator for ISO images using a Raspberry Pi Zero W and a python script https://github.com/tjmnmk/gadget_cdrom

Use cases:

- Setting up a fresh install of an OS on a computer without the need to create a bootable USB
- Running a live OS without the need to create a bootable USB
- Restoring an OS with Live CD tools
- Recovering a corrupted bootloader with boot-repair-disk-64bit.iso

What's the difference between my build and the pre-compiled image at https://github.com/tjmnmk/gadget_cdrom/releases ?

- Fully manual build on the official release of the OS to avoid any potential security risks
- Most recent release of Raspberry Pi OS Lite

## Build pictures

| ![untitled-1.jpg](pictures/small/untitled-1.jpg) | ![untitled-2.jpg](pictures/small/untitled-2.jpg) | ![untitled-3.jpg](pictures/small/untitled-3.jpg) |
| :----------------------------------------------: | :----------------------------------------------: | :----------------------------------------------: |

## Parts list

- Raspberry Pi Zero W - https://www.waveshare.com/wiki/Raspberry_Pi_Zero_W
- 64Gb A1 U1 C10 microsd card
- USB A board with pogo pins
- Plexiglass case
- 1.3inch OLED HAT - https://www.waveshare.com/wiki/1.3inch_OLED_HAT

## Software

- Raspberry Pi OS Lite - https://www.raspberrypi.org/software/operating-systems/
- https://github.com/tjmnmk/gadget_cdrom

## Build

- Cut the GPIO pins as short as humanly possible
- Sift through the shims and spacers from the previous builds to find the right combination
- Assemble
- Download https://downloads.raspberrypi.com/raspios_lite_armhf/images/raspios_lite_armhf-2024-07-04/2024-07-04-raspios-bookworm-armhf-lite.img.xz
- Burn the image to the microsd card using Imager
  - `sudo apt install rpi-imager`
  - `rpi-imager`
- Mount the microsd card on the host computer

- Enable SSH

  - `sudo touch /media/$USER/bootfs/ssh`

- Add user

  - `echo "rpiuser:$(echo 'rpipassword' | openssl passwd -6 -stdin)" | sudo tee /media/$USER/bootfs/userconf`

- Enable modules

      - `echo "dtoverlay=dwc2" | sudo tee -a /media/$USER/bootfs/config.txt`
      - `echo "dwc2" | sudo tee -a /media/$USER/rootfs/etc/modules`

- Unmount, put the card into the rpi, connect a USB-Ethernet adapter to the Raspberry Pi, hook up the USB-Ethernet adapter to a power source, connect the adapter with Ethernet cable to a router for Internet access
- Boot the Raspberry Pi
- SSH into the Raspberry Pi

  - `ssh -o IdentitiesOnly=yes rpiuser@raspberrypi.lan` password `rpipassword`.

- At this point the commands are run on the Raspberry Pi

- Enable SPI and set the time zone for the SSL to work correctly

  - `sudo raspi-config`
    - Interface Options, SPI, Yes, Back
    - Localisation Options, Timezone, Asia, Shanghai, Finish, Reboot (or use `sudo dpkg-reconfigure tzdata`)

- (optional) Replace the original repository with Chinese mirrors if in China

  - ```
    deb http://archive.raspberrypi.com/debian/ bookworm main
    # Uncomment line below then 'apt-get update' to enable 'apt-get source'
    #deb-src http://archive.raspberrypi.com/debian/ bookworm main
    ```

  - ```
    sudo tee /etc/apt/sources.list.d/raspi.list <<EOF
    deb https://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ bookworm main non-free contrib rpi
    deb-src https://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ bookworm main non-free contrib rpi
    # deb [arch=arm64] https://mirrors.tuna.tsinghua.edu.cn/raspbian/multiarch/ bookworm main
    EOF
    ```

  - `sudo sed -i '1s/^/#/' /etc/apt/sources.list`

- Create a larger swap file

  - `sudo swapoff -a`
  - `sudo dd if=/dev/zero of=/swap bs=1M count=1024`
  - `sudo mkswap /swap`
  - `sudo chmod 600 /swap`
  - `sudo swapon /swap`
  - `free -m`
  - `echo "/swap none swap sw 0 0" | sudo tee -a /etc/fstab`
  - `swapon --show`

- Update the Raspberry Pi

  - `sudo apt update`
  - `sudo apt upgrade`

- Install the required packages

  - `sudo apt install -y lm-sensors nmon screen git p7zip-full python3-rpi.gpio python3-smbus python3-spidev python3-numpy python3-pil fonts-dejavu ntfs-3g`

- Apply kernel patch for large ISOs, recompile natively and install the kernel. Takes a few hours, cross-compiling might be a better idea.

  - `sudo apt install bc bison flex libssl-dev make ca-certificates`
  - `screen`
  - `cd ~`
  - `git clone --depth=1 https://github.com/raspberrypi/linux`
    - or
      - `wget https://github.com/raspberrypi/linux/archive/refs/heads/rpi-6.6.y.zip`
      - `unzip rpi-6.6.y.zip`
      - `mv linux-rpi-6.6.y linux`
  - `cd linux`
  - `KERNEL=kernel`
  - `head Makefile -n 4`

    - ```

      # SPDX-License-Identifier: GPL-2.0

      VERSION = 6
      PATCHLEVEL = 6
      SUBLEVEL = 56

      ```

  - `cp drivers/usb/gadget/function/storage_common.c drivers/usb/gadget/function/storage_common.c.updated`
  - create a patch
    - `nano drivers/usb/gadget/function/storage_common.c.updated`
    - remove six lines at line 243
    - might as well create a patch while you're at it
      - `diff -Naru drivers/usb/gadget/function/storage_common.c.updated drivers/usb/gadget/function/storage_common.c > 00-remove_iso_limit.patch`
  - or just use the included patch for version 6.6.56
    - `patch drivers/usb/gadget/function/storage_common.c 00-remove_iso_limit.patch`
  - `make bcmrpi_defconfig`
  - `make -j6 zImage modules dtbs`
  - `sudo make -j6 modules_install`
  - `sudo cp /boot/firmware/$KERNEL.img /boot/firmware/$KERNEL-backup.img`
  - `sudo cp arch/arm/boot/zImage /boot/firmware/$KERNEL.img`
  - `sudo cp arch/arm/boot/dts/broadcom/*.dtb /boot/firmware/`
  - `sudo cp arch/arm/boot/dts/overlays/*.dtb* /boot/firmware/overlays/`
  - `sudo cp arch/arm/boot/dts/overlays/README /boot/firmware/overlays/`
  - `sudo reboot`

- Shutdown the Pi, remove the microsd card, mount it on the host computer, make backup

  - `cd ~`
  - `sudo dd if=/dev/mmcblk0 | gzip -9 > cdemu-backup.img.gz`
    - restore later with `sudo zcat cdemu-backup.img.gz | sudo dd of=/dev/mmcblk0` if needed

- Install the gadget_cdrom

  - `cd /opt`
  - `sudo git clone https://github.com/tjmnmk/gadget_cdrom.git`
  - `cd gadget_cdrom`
  - `sudo ./create_image.sh`, 40GB, fat32
  - `sudo ln -s /opt/gadget_cdrom/gadget_cdrom.service /etc/systemd/system/gadget_cdrom.service`
  - `sudo systemctl enable gadget_cdrom.service`
  - `sudo reboot`

## References

- Gadget CD-ROM Python script https://github.com/tjmnmk/gadget_cdrom
- github errors https://github.com/orgs/community/discussions/134430#discussioncomment-10226666
- Kernel compilation https://www.raspberrypi.com/documentation/computers/linux_kernel.html
