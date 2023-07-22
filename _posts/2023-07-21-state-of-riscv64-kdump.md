---
layout: post
title: "State of kdump on RISCV-64 openSUSE Tumbleweed"
date: "2023-07-21"
tags: [kdump]
description:
  The [Tumbleweed port for RISC-V](https://en.opensuse.org/openSUSE:RISC-V) is
  officially brok^Wstill under development. This post describes the state of
  kdump as of July 2023.
---

**NOTE:** This post is not suitable for anyone who does not want to
participate in fixing stuff. Read at your own peril.

## Reserve Memory for the Crash Kernel

Since there is no `kdump` package yet, everything must be set up manually.
The good news is that the Linux kernel supports the `crashkernel` command line
option on RISC-V. The bad news is that the correct size must be guessed, and
there is no clear guidance. Let's make it a big reservation and use
`crashkernel=512M`; half a gigabyte ought to be enough for anybody, right?

Add the `crashkernel` option to `/etc/default/grub`, regenerate `grub.conf` and
reboot. Check the kernel messages. You should see something like this:

```
riscv64-tumbleweed:~ # dmesg | grep crash
[    0.000000] crashkernel: reserved 0x00000000d7400000 - 0x00000000f7400000 (512 MB)
```

## Load the Panic Kernel

Next challenge is loading the panic kernel. Or any kernel with kexec, for that
matter. Upstream `kexec-tools` does not have any support for RISC-V. However,
the Tumbleweed package is built with an additional patch to enable this
architecture.

Now, there are generally two methods to load a panic kernel. Both are broken
for RISC-V.

### Load with the kexec_file_load(2) syscall

Since Linux 5.19, there is a kernel loader for RISC-V ELF images. Great! Let's
try it out:

```
riscv64-tumbleweed:~/kexec-tools-2.0.25 # kexec -slp /boot/Image
Warning: No cmdline or append string provided
Warning: No dtb provided, using /sys/firmware/fdt
Cannot determine the file type of /boot/Image
```

Ah, `kexec-tools` cannot load the boot image on RISC-V. Luckily,
`kernel-default` also contains a compressed ELF file. Let's try that:

```
riscv64-tumbleweed:~ # xz -dc /usr/lib/modules/`uname -r`/vmlinux.xz > vmlinux
riscv64-tumbleweed:~ # kexec -slp vmlinux
Warning: No cmdline or append string provided
Warning: No dtb provided, using /sys/firmware/fdt
kexec_file not supported on this architecture
Cannot load vmlinux
```

Not supported? But isn't `kexec-tools` patched for RISC-V? OK, time to look at
the source code (you may want to write a script to do the following; I'm too
lazy):

```
riscv64-tumbleweed:~ # osc co `rpm -q --qf %{disturl} kexec-tools`
[snip]
riscv64-tumbleweed:~ # cd openSUSE:Factory:RISCV/kexec-tools
riscv64-tumbleweed:~/openSUSE:Factory:RISCV/kexec-tools # quilt setup kexec-tools.spec
### rpmbuild: p
### md5sum: ......
### rpmbuild: ppp
riscv64-tumbleweed:~/openSUSE:Factory:RISCV/kexec-tools # cd kexec-tools-2.0.26
riscv64-tumbleweed:~/openSUSE:Factory:RISCV/kexec-tools # quilt push -a
```

The error message is found in `kexec/arch/riscv/kexec-elf-riscv.c` (BTW great
idea with breaking the error message across two lines, so you can have more
fun searching it):

```
int elf_riscv_load(int argc, char **argv, const char *buf, off_t len,
                   struct kexec_info *info)
{
	/* some declarations here */

	if (info->file_mode) {
		fprintf(stderr, "kexec_file not supported on this "
						"architecture\n");
		return -EINVAL;
	}
```

Ah. So the patch does not implement this method. To implement it, you can
change the code to look more like this:

```
	if (info->file_mode) {
		if (arch_options.initrd_path) {
			info->initrd_fd = open(arch_options.initrd_path, O_RDONLY);
			if (info->initrd_fd == -1) {
				fprintf(stderr,
						"Could not open initrd file %s:%s\n",
						arch_options.initrd_path, strerror(errno));
				return EFAILED;
			}
		}

		if (arch_options.cmdline) {
			info->command_line = (char *)arch_options.cmdline;
			info->command_line_len =
				strlen(arch_options.cmdline) + 1;
			}

		return 0;
	}
```

Don't forget to add the syscall number to `kexec/kexec-syscall.h`:

```
#ifdef __riscv
#define __NR_kexec_file_load    294
#endif
```

Recompile with the above changes, and, yahoo, it fails differently now:

```
riscv64-tumbleweed:~ # kexec -slp vmlinux
Warning: No cmdline or append string provided
Warning: No dtb provided, using /sys/firmware/fdt
syscall kexec_file_load not available.
```

This must be a rare gem among error messages. Sadly, something useful was
still logged by the kernel:

```
riscv64-tumbleweed:~ # dmesg | grep kexec_image
[ 6489.995627] kexec_image: The entry point of kernel at 0xd7400000
[ 6489.999103] kexec_image: Unknown rela relocation: 19
[ 6489.999322] kexec_image: Error loading purgatory ret=-8
```

Relocation number 19 is `R_RISCV_CALL_PLT`, and the fix is trivial. Torsten
has even [submitted a
patch](https://lore.kernel.org/lkml/20230310182726.GA25154@lst.de/). It's
stuck on some formalities, but it applies cleanly, so feel free to rebuild
your kernel in addition to `kexec-tools`.

Boot your new kernel and… Of course, `kexec` fails the same way, but the
kernel messages are slightly different:

```
riscv64-tumbleweed:~ # dmesg | grep kexec_image
[  281.658640] kexec_image: The entry point of kernel at 0xd7400000
[  281.664897] kexec_image: Unknown rela relocation: 20
[  281.665117] kexec_image: Error loading purgatory ret=-8
```

Relocation number 20 is `R_RISCV_GOT_HI20`. I haven't found any patches
floating around, so here's your unique chance to become a reputable kernel
hacker!

### Load with the kexec_load(2) syscall

Meanwhile, let's have a look at the other (older) method:

```
riscv64-tumbleweed:~ # kexec -clp vmlinux
Warning: No cmdline or append string provided
Warning: No dtb provided, using /sys/firmware/fdt
```

Looks good, but not very useful without an initrd. For now, let's just use the
standard one:

```
riscv64-tumbleweed:~ # kexec -clp vmlinux --initrd=/boot/initrd --reuse-cmdline
Warning: No dtb provided, using /sys/firmware/fdt
Could not find a free area of memory of 0x2000 bytes...
locate_hole failed
```

Er, not very informative beyond the unfortunate fact that the load failed.
Well, retry with `--debug`:

```
riscv64-tumbleweed:~ # kexec --debug -clp vmlinux --initrd=/boot/initrd --reuse-cmdline
Warning: No dtb provided, using /sys/firmware/fdt
Try gzip decompression.
kernel: 0xffffff7cac5010 kernel_size: 0xcab44f8
Got memory node at depth: 1
: #address-cells:2 #size-cells:2
Got region with 1 entries: memory@80000000
Got 1 /memory nodes
: #address-cells:2 #size-cells:2
reserved-memory: #address-cells:2 #size-cells:2
: #address-cells:2 #size-cells:2
Got region with 1 entries: mmode_resv0@80000000
Memory regions:
        0x80000000 - 0x8007ffff : RANGE_RESERVED (1)
        0x80080000 - 0xffffffff : RANGE_RAM (0)
Got ELF with total memsz 29612KB
Base paddr: 0x0, start_addr: 0x0
New base paddr for the ELF: 0xD7400000
New entry point for the ELF: 0xD7400000
kernel symbol _text vaddr = ffffffff80002000
kernel symbol _sinittext vaddr = ffffffff80a00000
page_offset:   ffffffff809ff000
get_crash_notes_per_cpu: crash_notes addr = fffb3000, size = 408
Elf header: p_type = 4, p_offset = 0xfffb3000 p_paddr = 0xfffb3000 p_vaddr = 0x0 p_filesz = 0x198 p_memsz = 0x198
get_crash_notes_per_cpu: crash_notes addr = fffd1000, size = 408
Elf header: p_type = 4, p_offset = 0xfffd1000 p_paddr = 0xfffd1000 p_vaddr = 0x0 p_filesz = 0x198 p_memsz = 0x198
vmcoreinfo header: p_type = 4, p_offset = 0xfe740000 p_paddr = 0xfe740000 p_vaddr = 0x0 p_filesz = 0x1024 p_memsz = 0x1024
Kernel text Elf header: p_type = 1, p_offset = 0x80202000 p_paddr = 0x80202000 p_vaddr = 0xffffffff80002000 p_filesz = 0x1f7e8cf p_memsz = 0x1f7e8cf
Elf header: p_type = 1, p_offset = 0x80080000 p_paddr = 0x80080000 p_vaddr = 0xa7f000 p_filesz = 0x57380000 p_memsz = 0x57380000
Elf header: p_type = 1, p_offset = 0xf7400000 p_paddr = 0xf7400000 p_vaddr = 0x77dff000 p_filesz = 0x8bfffff p_memsz = 0x8bfffff
load_elfcorehdr: elfcorehdr 0xf73ff000-0xf73ff3ff
chosen: #address-cells:2 #size-cells:2
memory@80000000: #address-cells:2 #size-cells:2
dtb_set_initrd: start 4045778944, end 4148160084, size 102381140 (99981 KiB)
Base addr for initrd image: 0xF125B000
Could not find a free area of memory of 0x2000 bytes...
locate_hole failed
```

That's slightly better; even much better when combined with the source code.
The problematic part is in `kexec/arch/riscv/kexec-riscv.c`:

```
	else if (arch_options.initrd_path) {
		/* something uninteresting */
		initrd_base = add_buffer_phys_virt(info, initrd_buf,
										   initrd_size,
										   initrd_size, 0,
										   min_usable,
										   max_usable, -1, 0);
		/* something uninteresting */
		min_usable = initrd_base;
	}

	/* Add device tree */
	add_buffer_phys_virt(info, fdt->buf, fdt->size, fdt->size, 0,
						 min_usable, max_usable, -1, 0);
```

Here, the `-1` argument to `add_buffer_phys_virt()` means: “Allocate at the
top of the range.” If the minimum base is then set to the returned address, no
wonder that there is no hole left for the device tree blob.

All that you need is replace `min_usable` with `max_usable`. Rebuild and
enjoy.

As for fixing the upstream code, I'm on it. The original patch was sent to the
[kexec mailing list](https://lists.infradead.org/mailman/listinfo/kexec) in
October 2022,
[here](http://lists.infradead.org/pipermail/kexec/2022-October/026057.html),
and not much has happened since then.

## Crash the Kernel

Assuming you have applied the fix, you can load the panic kernel now:

```
riscv64-tumbleweed:~ # kexec -clp vmlinux --initrd=/boot/initrd --reuse-cmdline"
Warning: No dtb provided, using /sys/firmware/fdt
```

You can verify that the kernel is indeed loaded:

```
riscv64-tumbleweed:~ # cat /sys/kernel/kexec_crash_loaded
1
```

Everything ready to crash the kernel!

```
riscv64-tumbleweed:~ # echo c > /proc/sysrq-trigger
```

If you have a serial console, you'll see a farewell message from the crashed
kernel:

```
[ 3399.306279][  T758] sysrq: Trigger a crash
[ 3399.322014][  T758] Kernel panic - not syncing: sysrq triggered crash
[ 3399.323723][  T758] CPU: 1 PID: 758 Comm: bash Kdump: loaded Tainted: G            E      6.5.0-rc2-ptesarik-dirty #3 6ac4b277903261e1d98e5903e1c20025abd5b13c
[ 3399.325984][  T758] Hardware name: riscv-virtio,qemu (DT)
[ 3399.327215][  T758] Call Trace:
[ 3399.328616][  T758] [<ffffffff8000682c>] dump_backtrace+0x28/0x30
[ 3399.330643][  T758] [<ffffffff80950e34>] show_stack+0x38/0x44
[ 3399.331439][  T758] [<ffffffff8095c964>] dump_stack_lvl+0x44/0x5c
[ 3399.332340][  T758] [<ffffffff8095c994>] dump_stack+0x18/0x20
[ 3399.333095][  T758] [<ffffffff80951170>] panic+0x10a/0x2f2
[ 3399.333853][  T758] [<ffffffff8062e59a>] sysrq_reset_seq_param_set+0x0/0x76
[ 3399.334741][  T758] [<ffffffff8062ed74>] __handle_sysrq+0x90/0x180
[ 3399.335522][  T758] [<ffffffff8062f486>] write_sysrq_trigger+0x7a/0x94
[ 3399.336347][  T758] [<ffffffff80347d46>] proc_reg_write+0x48/0x96
[ 3399.337157][  T758] [<ffffffff802c3d98>] vfs_write+0xc4/0x31c
[ 3399.337935][  T758] [<ffffffff802c42ee>] ksys_write+0x64/0xe8
[ 3399.338689][  T758] [<ffffffff802c438c>] sys_write+0x1a/0x22
[ 3399.339514][  T758] [<ffffffff8095d4a6>] do_trap_ecall_u+0x140/0x154
[ 3399.340446][  T758] [<ffffffff80003c14>] ret_from_exception+0x0/0x64
[ 3399.343286][  T758] SMP: stopping secondary CPUs
[ 3399.345788][  T758] Starting crashdump kernel...
[ 3399.346781][  T758] Will call new kernel at d7400000 from hart id 1
[ 3399.347607][  T758] FDT image at f1259000
[ 3399.348209][  T758] Bye...
```

And that's it. The new kernel never starts. :-(

This system ran in QEMU, so I was able to attach to it with `gdb`, look
around, set some breakpoints, restart, and so on. After a lot of head
scratching, I believe I was loading the wrong ELF file. The packaged `vmlinux`
file is the one which is found in the kernel build root directory, and it does
not contain relocation data.

This might be a bug in `binutils`, but that's out of scope for me. The
important thing is that there is also a `vmlinux.relocs` file in the build
root directory, and this one does contain the relocation data. If you skipped
the kernel recompile exercise above, now is the time to catch up.

The only downside is that this file is a tad too big, especially since `kexec`
will load the full content into memory:

```
-rwxr-xr-x 1 petr petr 1.2G Jul 21 13:41 build/vmlinux.relocs
```

But most of it is not needed, so here goes the magic:

```
petr@meshulam:~/riscv/linux> objcopy -R .note -R .note.gnu.build-id -R .comment -S vmlinux.relocs vmlinux.stripped
petr@meshulam:~/riscv/linux> l -h vmlinux.stripped
-rwxr-xr-x 1 petr petr  28M Jul 21 13:55 build/vmlinux.stripped
```

Now, if you load this `vmlinux.stripped`, the kernel finally boots! Beware,
the system looks normal, but it runs with only 512M of memory.

## Get the Core Dump

Time to log in and look for a core dump, right? The good news is that the new
kernel created a `vmcore` file:

```
riscv64-tumbleweed:~ # ls -l /proc/vmcore
-r-------- 1 root root 1645350912 Jul 21 13:48 /proc/vmcore
```

The bad news is that something is broken when reading it. I tried to transfer
it with `rsync` and I crashed the panic kernel (100% reproducible):

```
[ 1558.887710][  T929] usercopy: Kernel memory exposure attempt detected from SLUB object 'kmem_cache_node' (offset 0, size 4096)!
[ 1558.894365][    C1] ------------[ cut here ]------------
[ 1558.894460][    C1] kernel BUG at mm/usercopy.c:102!
[ 1558.894738][    C1] Kernel BUG [#1]
[ 1558.894800][    C1] Modules linked in: af_packet(E) nls_iso8859_1(E) nls_cp437(E) vfat(E) fat(E) virtio_net(E) net_failover(E) failover(E) virtio_balloon(E) fuse(E) configfs(E) ip_tables(E) x_tables(E) ext4(E) mbcache(E) jbd2(E) virtio_blk(E) virtio_mmio(E) sg(E) efivarfs(E)
[ 1558.896084][    C1] CPU: 1 PID: 929 Comm: rsync Tainted: G            E      6.5.0-rc2-ptesarik #1 ea1239917c1ac9a4177d76230dd8750c5dd304d4
[ 1558.896358][    C1] Hardware name: riscv-virtio,qemu (DT)
[ 1558.896464][    C1] epc : usercopy_abort+0x78/0x7a
[ 1558.897110][    C1]  ra : usercopy_abort+0x78/0x7a
[ 1558.897152][    C1] epc : ffffffff80954c82 ra : ffffffff80954c82 sp : ff20000000213bf0
[ 1558.897180][    C1]  gp : ffffffff81b59338 tp : ff60000000745640 t0 : 79706f6372657375
[ 1558.897199][    C1]  t1 : 0000000000000075 t2 : 3a79706f63726573 s0 : ff20000000213c10
[ 1558.897251][    C1]  s1 : 0000000000000000 a0 : 000000000000006b a1 : ff6000007ffba700
[ 1558.897271][    C1]  a2 : ff6000007ffc7328 a3 : 0000000000000000 a4 : 0000000000000000
[ 1558.897290][    C1]  a5 : 0000000000000000 a6 : ffffffff81b7c368 a7 : 0000000000000038
[ 1558.897309][    C1]  s2 : 0000000000001000 s3 : ff60000000401100 s4 : 0000000000000001
[ 1558.897328][    C1]  s5 : 0000000000000000 s6 : ff20000000213d30 s7 : 0000000000000000
[ 1558.897347][    C1]  s8 : ffffffff81be7880 s9 : 0000000000001000 s10: ffffffff81be7830
[ 1558.897368][    C1]  s11: 0000000000000000 t3 : ffffffff81e95787 t4 : ffffffff81e95787
[ 1558.897386][    C1]  t5 : ffffffff81e95788 t6 : ff20000000213a08
[ 1558.897403][    C1] status: 0000000200000120 badaddr: 0000000000000000 cause: 0000000000000003
[ 1558.897545][    C1] [<ffffffff80954c82>] usercopy_abort+0x78/0x7a
[ 1558.897676][    C1] [<ffffffff8028f8ce>] __check_heap_object+0x120/0x156
[ 1558.897700][    C1] [<ffffffff802bd238>] __check_object_size+0x30e/0x3ca
[ 1558.897717][    C1] [<ffffffff8000c44c>] copy_oldmem_page+0x5e/0x9a
[ 1558.897735][    C1] [<ffffffff8035a546>] read_from_oldmem.part.0+0xf0/0x14c
[ 1558.897755][    C1] [<ffffffff8035a75e>] read_vmcore+0x118/0x37a
[ 1558.897772][    C1] [<ffffffff803479be>] proc_reg_read_iter+0x48/0x90
[ 1558.897789][    C1] [<ffffffff802c36ea>] vfs_read+0x1c0/0x258
[ 1558.897807][    C1] [<ffffffff802c41e4>] ksys_read+0x64/0xe8
[ 1558.897825][    C1] [<ffffffff802c4282>] sys_read+0x1a/0x22
[ 1558.897861][    C1] [<ffffffff8095d4a6>] do_trap_ecall_u+0x140/0x154
[ 1558.897898][    C1] [<ffffffff80003c14>] ret_from_exception+0x0/0x64
[ 1558.898341][    C1] Code: e036 86aa 7517 00b6 0513 0d65 d097 ffff 80e7 2140 (9002) 1141 
[ 1558.912429][    C1] ---[ end trace 0000000000000000 ]---
[ 1558.912607][    C1] Kernel panic - not syncing: Fatal exception in interrupt
[ 1558.912702][    C1] SMP: stopping secondary CPUs
```

Of course, you can use `hardened_usercopy=n` on the kernel command line to
disable the hardening, but I'm not sure it is the right solution.
