---
layout: post
title: "Plot Twist in the Sandbox Story"
date: "2023-12-15"
tags: [sandbox, kernel]
description:
  If you follow my random posts, you may wonder what happened to the Sandbox
  Mode project. The goals have not changed, but the path to get there might
  be quite different.
---

## What Happened?

The original plan was to provide at least one in-tree user. However, the
minimum implementation was too limited to allow converting any existing code,
unless it was so trivial that it made little sense to run it in a sandbox.

So, the patch series kept growing... It is 31 patches now (and counting), they
are still not quite fit to make a sensible conversion of existing code, and I
can't be even sure that the whole idea does not get NAKed right away by an
influential kernel maintainer.

At this point I had a very fruitful discussion with Huawei's Roberto Sassu how
to get at some results, and most importantly how to get some feedback from the
community.

## The State

There has been good progress on sandbox mode (SBM) features:

* The public API looks reasonably easy to use.
* Sandbox code runs with CPL == 3 on x86_64.
* Page faults terminate the sandbox, returning `-EFAULT` from `sbm_call()`.
* Sandbox mode can be preempted.
* Spectre mitigations (such as retpolines) work as intended.
* Library functions like memcpy() or strcpy() are available.
* Dynamic memory allocation will grow the sandbox.

With all the above, I was able to run several decompressors inside a sandbox
to verify that the idea works for some real-world workloads.

However, the patch series is too complex for review, and there are still known
issues.

## The Plan

Instead of making a complete series and converting some existing code, the
plan is now to submit a very **minimal** series to elicit some feedback from
the community. This series is essentially just the public API and a trivial
“bounce-buffer” implementation. But it is **complete**. It does include
documentation and even a KUnit test case. It hopefully passes internal review
at Huawei and will be posted early next week.

## Other Random Remarks

You may not know that KUnit test cases can run in QEMU or as user-mode Linux
(UML). The latter is the default. Now, my sandbox mode KUnit test suite ran
fine when built for x86_64, but UML ran into an infinite loop. It took half a
day of debugging. And man, is this code broken!

If interested, see my patches
[here](https://lore.kernel.org/linux-um/20231215121431.680-1-petrtesarik@huaweicloud.com/T/).
