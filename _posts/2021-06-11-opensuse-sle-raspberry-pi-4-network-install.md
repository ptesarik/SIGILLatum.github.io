---
layout: post
title: "openSUSE/SLE Network Install on Raspberry Pi 4"
date: "2021-06-11"
tags: [guide, raspberry, rpi4, dhcp, tftp, install]
description:
  This guide describes how to install a Raspberry Pi from the network,
  i.e. without writing any image to the SD card first. It shows how to load the
  Raspberry Pi firmware and the operating system from another server over an
  IPv4 network. It does not cover setting up network storage for a diskless
  Raspberry Pi.
---

## Prepare Your Raspberry Pi

Network boot is disabled in the default (factory) configuration of your
Raspberry Pi 4. To allow booting from the network, the EEPROM bootloader
configuration must be changed. To do this, prepare a “recovery” SD card:

### The Easy Way

- Install the
  [Raspberry Pi Imager](https://snapcraft.io/install/rpi-imager/opensuse)
  on your desktop system.

- In the Imager, choose _Operating system_ 〉 _Bootloader_ 〉 _Network
  boot_. Select the microSD card where to write the image.

- The rest of the process is identical with the instructions from the EEPROM
  recovery `README.txt`:

  - Make sure your Raspberry Pi is powered off.

  - Insert the microSD card into your Raspberry Pi.

  - Power on your Raspberry Pi.

  - Wait at least 10 seconds.

Note that BOOT_ORDER is set to `0xf21`, i.e. it will attemp the microSD card
first, and only if that fails, it will boot from network. Read the next
section if you need more control.

### The Hard Way

- Download the latest release of the
  [EEPROM bootloader](https://github.com/raspberrypi/rpi-eeprom/releases/latest).

  Note that the recovery ZIP file does not contain the `rpi-eeprom-config`
  script. Download it separately, e.g. like this:

      wget https://raw.githubusercontent.com/raspberrypi/rpi-eeprom/master/rpi-eeprom-config

  Alternatively, you can download the source ZIP or tar.gz archive, which
  includes the script and multiple firmware files. The oldest firmware version
  covered by this guide is from **October 8 2019**.

- Unzip the recovery ZIP to a blank microSD card with a formatted FAT
  partition.

- Modify the factory settings:

  - Extract the configuration file from the firmware:

        python3 rpi-eeprom-config pieeprom.bin --out bootconf.txt

  - Edit `bootconf.txt` and change the `BOOT_ORDER` variable. To configure
    network boot, set the value to `0x2`, or `0xf2` to keep trying forever if
    the boot attempt fails. A detailed explanation of this option can be found
    in the
    [official documentation](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2711_bootloader_config.md#BOOT_ORDER).

  - Create an updated firmware image and remove the original file:

        python3 rpi-eeprom-config pieeprom.bin --config bootconf.txt --out pieeprom.upd
        rm pieeprom.bin

  - Update the checksum file:

        sha256sum pieeprom.upd | cut -c 1-64 > pieeprom.sig

- Unmount the microSD card.

- Follow the steps from the EEPROM recovery `README.txt`.

At this point, your Raspberry Pi is ready to boot over Ethernet. You can
remove the microSD card now, unless you are going to install the operating
system on it, of course.

## Server Overview

Depending on your existing infrastructure, you may want to use up to three
separate systems:

1. DHCP server
2. TFTP server
3. HTTP or FTP server

This guide assumes that they are three different hosts in a private network:

1. `dhcp` (192.168.0.100)
2. `tftp` (192.168.0.101)
3. `inst` (192.168.0.102)

It is perfectly fine to run any two or all of these services on a single
host. Adjust the following instructions accordingly.

## DHCP Server Configuration

This guide assumes that you already have a working DHCP server in your
network, it can properly configure IPv4 hosts, and you only want to add the
Raspberry Pi network boot extras.

The DHCP server is queried multiple times during boot: first by the EEPROM
bootloader, and then a couple of times by U-Boot. There are two ways to
configure the DHCP server.

### Without the Pseudo-PXE Option

Given the network setup described above, set up your DHCP server to send:

- **(for EEPROM)** Option 66 (TFTP Server): `192.168.0.101`

- **(for U-Boot)** _siaddr_ header field: `192.168.0.101`

- **(for U-Boot)** Option 67 (Bootfile name): `EFI/BOOT/bootaa64.efi`

  The path corresponds to the location of UEFI GRUB on `tftp` relative to the
  TFTP root directory.

With _dnsmasq_, all these options can be configured on a single line:

    dhcp-boot=EFI/BOOT/bootaa64.efi,192.168.0.101,192.168.0.101

### With the Pseudo-PXE Option

If you do not want to hardcode the IP address of `tftp` in your DHCP
configuration, you probably have to use the pseudo-PXE option. Let your DHCP
server send:

- **(for EEPROM)** Option 60 (Class Identifier) and Option 43 (Vendor specific
  information): Provide valid Raspberry Pi 4 [pseudo-PXE options]({% post_url 2021-06-11-raspberry-pi-4-network-boot %}#raspberry-pi-firmware).

- **(for both)** _siaddr_ header field: `192.168.0.101`

- **(for U-Boot)** Option 67 (Bootfile name): `EFI/BOOT/bootaa64.efi`

Add these options for _dnsmasq_:

    pxe-service=x86PC,"Raspberry Pi Boot",32768,tftp
	dhcp-boot=EFI/BOOT/bootaa64.efi,tftp,tftp

### Client Tagging

If your DHCP server serves other clients besides Raspberry Pis, then you
probably want to send the above options only to Raspberry Pi 4 clients. They
can be recognized by their MAC address. For example, put this into your
_dnsmasq_ configuration file:

    dhcp-mac=set:rpi4,dc:a6:32:*:*:*

And then prepend "`tag:rpi4,`" (including the comma) to all the other option
values.

## Installation Files

Download the installation ISO. This guide refers to the online ISO, but you
can download and use the full ISO instead, e.g. if your Raspberry Pi has no
connection to the public Internet, or if you plan to install more than one.

The installation image is a hybrid image, and it contains two partitions. Both
are needed, but loop-mounting the installation image file will let you access
only the ISO9660 filesystem. Instead, set up a partitioned loopback device on
the `tftp` server:

    losetup --show -fP SLE-15-SP2-Online-aarch64-GM-Media1.iso

Now mount the EFI partition:

    mount -r /dev/loop0p1 /mnt

Change to the TFTP root directory:

    cd /srv/tftpboot

If this TFTP server is used only for Raspberry Pi installations, you can copy
the Raspberry Pi firmware (`start4.elf`), all firmware-related files and the
EFI binaries into the TFTP root:

    cp -r /mnt/* .

You may prefer to put these files into a per-product subdirectory and create a
symbolic link to the serial number of your Raspberry Pi, e.g.:

    cp -r /mnt SLE-15-SP2-aarch64
	ln -s SLE-15-SP2-aarch64 af04b2da

If you choose this arrangement, make sure the product subdirectory is also
included in your boot file name (DHCP option 67).

You can unmount the EFI partition now:

    umount /mnt

Next, copy additional GRUB files, the installation kernel and initial ramdisk
from the second ISO image partition:

    mount -r /dev/loop0p2 /mnt
    cp -r /mnt/boot .
    umount /mnt

The loop device is no longer needed on `tftp`, so clean it up:

    losetup -d /dev/loop0

Finally, make data on the ISO9660 partition available to your HTTP server on
`inst`. For example:

    cd /srv/www/htdocs
    mkdir SLE-15-SP2-aarch64
	mount -o loop,ro /path/to/SLE-15-SP2-Online-aarch64-GM-Media1.iso SLE-15-SP2-aarch64

It is usually a good idea to edit `EFI/BOOT/grub.cfg` on the `tftp` server and
set up the installation source, so you do not have to enter it manually in the
installer. Find the `menuentry` for _Installation_ and add your options to the
`linux` command. For example, the following line would set up a serial
console, get the installation media via HTTP from `inst` and allow to start
the installation over SSH:

    linux /boot/aarch64/linux console=ttyS0,115200 install=http://inst/SLE-15-SP2-aarch64 ssh=1 sshpassword=foobar123

## Starting the Installation

You can now power on your Raspberry Pi and wait until the installation system
is ready.

Well, almost ready.

The Raspberry Pi does not have a hardware clock, and the installation system
does not use NTP. As a result, the system clock is not set correctly, which
may cause trouble with TLS connections (e.g. to the SUSE Customer Center). Set
the system clock manually before starting the installation (assuming your
Raspberry Pi 4 is known in your network under the host name `rpi4`):

    ssh root@rpi4 "date --set=@$( date +%s)"
