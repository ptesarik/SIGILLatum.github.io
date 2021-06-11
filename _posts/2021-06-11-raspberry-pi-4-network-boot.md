---
layout: post
title: "Raspberry Pi 4 Network Boot"
date: "2021-06-11"
tags: [raspberry, rpi4, boot, dhcp, tftp]
description:
  Find out about all the gory details of booting a Raspberry Pi 4 over an
  Ethernet connection, including openSUSE/SLE specifics.
---

The Raspberry Pi first loads the VPU firmware, then the ARM bootloader (or
directly the operating system). Both stages can be booted over Ethernet using
TFTP if the bootloader configuration option `BOOT_ORDER` includes a NETWORK
mode (value `0x2`). Consult the
[Raspberry Pi 4 bootloader documentation](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2711_bootloader_config.md)
for more details.

## Raspberry Pi Firmware

First, let us explain how the Raspberry Pi EEPROM bootloader works:

- The TFTP server address is taken from one of these places (in order of
  preference):

  - `TFTP_IP` bootloader config option (stored in EEPROM)
  - DHCP option 66 (TFTP Server Name)
  - IP address of the next boot server (_siaddr_ header field)
  - IP address of the DHCP server itself

  The last two alternatives are considered only if a valid Raspberry Pi
  pseudo-PXE option is found in the DHCP reply, where _valid_ means that:
  - DHCP option 60 (Class Identifier) is `PXEClient`,
  - DHCP option 43 (Vendor specific information) includes a PXE option 9
    (PXE_MENU), with the magic text `Raspberry Pi Boot`.

  DHCP Option 66 does **not** require these pseudo-PXE options, but there is
  another catch. This option should contain the _host name_ of the boot
  server. Since the Raspberry Pi bootloader does not implement a DNS
  resolver, the host name must be in fact an IPv4 address in dotted quad
  notation.

- The TFTP file name is hardcoded as `start4.elf`. The file name is prefixed
  with a subdirectory name. Depending on the `TFTP_PREFIX` bootloader config
  option, the prefix is:

  - **0** (default): board serial number,
  - **1**: any string specified in `TFTP_PREFIX_STR`,
  - **2**: device MAC address.

  If the boot file is not found using a prefix, the EEPROM bootloader also
  attempts to load it from the root directory.

- The EEPROM bootloader uses zero for option 93 (Client Architecture) in the
  DHCP request. This value is in fact incorrect; it is reserved for _Intel
  x86PC_. Here, the actual client architecture is BCM2711 VPU, but the
  Raspberry Pi uses zero, most likely because there is no reserved value for
  the BCM2711 VPU.

  In any case, this DHCP option can be used by the DHCP server to distinguish
  between a DHCP request made by the EEPROM bootloader and a request made by
  U-Boot, GRUB or a Linux DHCP client.

## U-Boot

Second, let us look at U-Boot. Unattended installation will use the built-in
configuration. The `distro_bootcmd` for openSUSE and SUSE follows this boot
order:

1. on-board microSD card reader,
2. USB mass storage,
3. network using U-Boot PXE,
4. network using UEFI PXE.

The TFTP server address is always taken from the _siaddr_ DHCP header
field. The file name is handled differently for U-Boot PXE and UEFI.

The U-Boot PXE method looks for a configuration file inside a `pxelinux.cfg`
directory, which must be in the same directory as the boot file (sent in the
_file_ DHCP header field). These names are tried for the PXE configuration
file:

- the client MAC address in the form `01-xx-xx-xx-xx-xx-xx` (where `01` means
  Ethernet)
- the hexadecimal IP address in all uppercase (e.g. 192.168.0.99 becomes
  `C0A80063`)
- an initial part of the IP address, i.e. each successive attempt removes one
  hexadecimal digit from the end
- `default-arm-bcm283x-rpi`
- `default-arm-bcm283x`
- `default-arm`
- `default`

With UEFI, The boot file name is taken from the _file_ header field, or
from DHCP option 67 (Bootfile name), depending on DHCP option 52 (Option
overload). This file should be the next-stage bootloader, i.e. GRUB.

To distinguish between the U-Boot PXE and UEFI requests on the DHCP server,
check the value of DHCP option 93 (Client Architecture):

- PXE is 22 (arm 64 uboot)
- UEFI is 11 (ARM 64-bit UEFI)

## GRUB

GRUB uses the network routines of U-Boot (through its EFI implementation), so
it does not ask the DHCP server again. It does load additional files from the
TFTP server (most importantly `grub.cfg`), but that's not specific to the
Raspberry Pi. Read GRUB documentation for details.
