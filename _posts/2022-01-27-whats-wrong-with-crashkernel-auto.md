---
layout: post
title: "What's Wrong with crashkernel=auto"
date: "2022-01-27"
tags: [kdump]
description:
  When setting up kernel dump, one of the tricky tasks is to
  reserve a sufficiently large portion of RAM for the crash
  kernel. This post explains why the kernel cannot really decide
  this automatically.
---

## The Problem

The “crash kernel” is a complete self-contained OS that is booted
upon a kernel crash. Usually, its sole function is to save an image
of the crashed kernel somewhere it can be analyzed later. Since it
saves the content of all memory used by the crashed kernel, it
cannot use that same memory for its own operation. That's why it
needs its own separate RAM region, not used by anything else.

So, how much RAM should be reserved for this crash kernel? Too much
and your precious RAM is effectively wasted. Too little and dumping
will fail on OOM. Besides, this is a large RAM region that must be
continuous in physical memory, so it must be reserved early at
boot, before memory gets too fragmented for such huge allocations.
In other words, its size must be known before you even load the
kernel. No wonder system admins are often frustrated by choosing a
suitable number.

## The Solution

Since the allocation happens at early boot time, the amount cannot
be determined by a user-space application, because it's already too
late when the first user-space process starts. Red Hat engineers
believe that they can solve it in the kernel, adding a
`crashkernel=auto` option. It is implemented like this:

```
	if (strncmp(ck_cmdline, "auto", 4) == 0) {
#if defined(CONFIG_X86_64) || defined(CONFIG_S390)
		ck_cmdline = "1G-4G:160M,4G-64G:192M,64G-1T:256M,1T-:512M";
#elif defined(CONFIG_ARM64)
		ck_cmdline = "2G-:448M";
#elif defined(CONFIG_PPC64)
		char *fadump_cmdline;

		fadump_cmdline = get_last_crashkernel(cmdline, "fadump=", NULL);
		fadump_cmdline = fadump_cmdline ?
				fadump_cmdline + strlen("fadump=") : NULL;
		if (!fadump_cmdline || (strncmp(fadump_cmdline, "off", 3) == 0))
			ck_cmdline = "2G-4G:384M,4G-16G:512M,16G-64G:1G,64G-128G:2G,128G-:4G";
		else
			ck_cmdline = "4G-16G:768M,16G-64G:1G,64G-128G:2G,128G-1T:4G,1T-2T:6G,2T-4T:12G,4T-8T:20G,8T-16T:36G,16T-32T:64G,32T-64T:128G,64T-:180G";
#endif
		pr_info("Using crashkernel=auto, the size chosen is a best effort estimation.\n");
	}
```

Ouch! This code hardwires some arbitrary numbers into the kernel
source… But even if it didn't, can the kernel do any better?

How much memory is _really_ needed? In other words, what must fit
into RAM while saving the dump? Essentially, a complete
single-purpose operating system. It usually consists of the
currently installed kernel and selected user-space binaries.
These days, the initrd image is usually created by
[dracut](https://dracut.wiki.kernel.org/index.php/Main_Page).

When booted after a crash, system RAM will contain:

* Kernel base code,
* Kernel modules,
* Kernel data structures,
* RAM-based filesystem (actually `rootfs`),
* User-space data.

None of the above is known when the original kernel reserves a
crash kernel area. The panic kernel and initrd is loaded by `kexec`
much later.

How should the original kernel know which files go into a crash
initrd, and how big those files will be? What user-space processes
will be running? Will that be systemd and a bunch of services, or
merely a custom shell script? Will that script save the dump to an
ext3 filesystem on an SD card, or to a remote NFSv4 share after
configuring its own IP address with DHCP? More RAM is definitely
needed in the latter case, right?

I have never found an easy way to estimate the size of kernel
run-time data, not even from within the kernel itself. It is known
to depend on a lot of things, memory size being only one of them.
AFAIK nobody can even make a complete list of those things.

Oh, and you may even use a different crash kernel than the
currently booting one.

I'm afraid, whatever guess can be made by the kernel at boot time,
will be good only for one specific hardware and software
configuration. But then it shouldn't be in the kernel code.
