---
layout: post
title: "kdump on 32-bit Arm"
date: "2023-07-13"
tags: [kdump]
description:
  How to get kdump going on openSUSE Tumbleweed for 32-bit Arm.
---

**NOTE:** This post is about openSUSE Tumbleweed only.

## Installation

AFAIK there are no ISO installation media, but there are plenty of pre-built
[images](http://download.opensuse.org/ports/armv7hl/tumbleweed/appliances/).
The kdump package is currently not built, and there are good reasons for
that. Read on.

## The UEFI Debacle

Arm QEMU VMs use AAVMF firmware by default. However, the 32-bit AAVMF firmware
cannot find any disks, rendering the system unbootable. This is a bug that
should be fixed, but I haven't had time to debug this any further.

I was able to boot the system with U-Boot, though. The QEMU image is not
packaged, but you can build it from sources with `make qemu_arm_defconfig`.
Install it to some convenient place; I installed mine as
`/usr/share/qemu/u-boot-qemu-arm.bin` and modified my libvirt VM configuration
thus:

	<os>
	  <type arch='armv7l' machine='virt-8.0'>hvm</type>
	  <loader readonly='yes' type='rom'>/usr/share/qemu/u-boot-qemu-arm.bin</loader>
	  <bootmenu enable='no'/>
	</os>

EFI variables are lost at that point, making the system fall back to the
default EFI executable on boot, which is missing. Mount your EFI partition and
copy `EFI/opensuse/grubarm.efi` to `EFI/BOOT/bootarm.efi`.

The system then boots fine with U-Boot's minimal UEFI implementation.

## Fighting with kexec-tools

Next, I added `crashkernel=128M` to the kernel command line, rebooted and
tried loading the panic kernel with a standard initrd:

	kexec -l -p /boot/zImage --initrd=/boot/initrd --append="console=ttyAMA0"`

Good, this seems to work. So, let's crash the kernel:

	echo c > /proc/sysrq-trigger

The kernel boots and spits messages on the serial console. Originally, I
thought that Arm kernels are not relocatable, because there is no
`CONFIG_RELOCATABLE`, but Arm can achieve similar results with
`CONFIG_AUTO_ZRELADDR` and `CONFIG_ARM_PATCH_PHYS_VIRT`.

Anyway, one of the kernel messages complains about _invalid magic at start of
compressed archive_ and eventually fails to mount root filesystem. That's
because the initramfs image is overwritten when the kernel uncompresses
itself. This is also a bug that should be fixed, but meanwhile you can
persuade kexec to load initrd higher in memory with
e.g. `--image-size=0x34c0000`.

## UEFI Strikes Again

The kernel can find the initramfs now, but it fails to initialize ELF core
headers (`elfcorehdr`) with a fat kernel warning that the kernel is trying to
remap normal RAM as an I/O region. This would indeed cause serious cache
aliasing issues on Arm. However, ELF core headers are allocated at the top of
crash kernel reserved region and is removed from usable memory by kexec-tools,
so this physical region should not be recognized as RAM by the panic kernel.

Hm. Turn on memblock debugging.

Ah. The panic kernel receives the unmodified EFI memory map and obediently
processes all `EFI_CONVENTIONAL_MEMORY` descriptors. No, this processing
cannot be turned off with `noefi`.

## How to Make it Work

Since the panic kernel won't work if the system was booted with UEFI, what
about booting directly from U-Boot? Reboot the system and interrupt U-Boot to
get the command line. Then boot Linux manually, e.g.:

	fdt addr $fdtcontroladdr
	fdt set /chosen bootargs "console=ttyAMA0 root=UUID=e370fb49-c0ea-4d6c-a788-727e11e87a81 rw crashkernel=128M"

My system has a separate /boot partition at vda2, so I load the kernel and
initrd from there:

	load virtio 0:2 $kernel_addr_r zImage-6.3.9-1-default
	load virtio 0:2 $ramdisk_addr_r initrd-6.3.9-1-default

Note the size of the ramdisk; that's the magic number below:

	bootz $kernel_addr_r $ramdisk_addr_r:16959751 $fdtcontroladdr

When the system boots, load the panic kernel (see above) and crash the
system. This time, you should get `/proc/vmcore` as expected.

You can also build and install `kdump` from the source repository. There are a
few missing bits at the moment:

- `init/mkdumprd` will fail to find a kernel, because the search list does not
  include `zImage`,
- same issue with `calibrate/run-qemu.py`
- unless your machine is an `armv8b` or `armv8l`, the QEMU program name will
  be incorrect
- QEMU requires additional parameters (e.g. `-machine virt`).
