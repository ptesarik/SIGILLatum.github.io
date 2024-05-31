---
layout: post
title: "Performance of Local IP Traffic"
date: "2024-05-31"
tags: [kernel, performance, network]
description:
  When it comes to local IP traffic, skb freeing deferral considered harmful.
---

As usual, this post is an attempt at sorting out my own ideas, but there are
some interesting bits for others.

## Background

As you may know, I returned to SUSE in April, this time as a kernel
performance engineer with focus on Arm. My first task was a regression between
SLE 15 SP5 (kernel 5.14) and SLE 15 SP6 (kernel 6.4) in UDP and TCP throughput
reported by a [netperf](https://hewlettpackard.github.io/netperf/) test case
as run by the SUSE standard benchmark (aka
[mmtests](https://github.com/gormanm/mmtests)). This regression was primarily
observed on Ampere Altra systems, but I believe all systems are affected to
some extent.

## Problem Statement

The test runs a server process and a client process. Both are single-threaded,
and the scheduler places them on two different CPUs. The client process sends
data as fast as possible; the server provides a data sink, receiving data with
no further processing. This is a plausible approximation of a real-world
scenario: clients tend to be simple (e.g. single-threaded), and servers which
accept large data chunks are designed to distribute data processing so that
the receiving thread can receive further data at full speed. This matches the
`mpstat` results on my system with SLE15 SP5. The client process CPU is fully
utilized, while the server process CPU is utilized to approx. 70%.

Now install a SLE15 SP6 kernel, reboot and restart the test. Throughput drops
by more than 10%. How much exactly is difficult to say, because results start
to vary a lot. So much that `netperf` usually complains that the “desired
confidence was not achieved within the specified iterations”, and suggests a
confidence interval of 15% or worse. This variance is also undesirable.

## Analysis

Looking at the differences between a SLE15 SP5 test run and a SLE15 SP6 run,
a few things stand out:

- The server process CPU is less utilized (around 60% down from SP5's 70%).
- The server process is never migrated to another CPU.
- More context switches are reported by `pidstat -C netserver -w`

This brings up a few questions.

First, why did the SP5 kernel migrate the server process? And how is that ever
good for performance? Let's use `trace-cmd` to trace
`sched:sched_migrate_task` and `sched:sched_wakup` events and see what happens
here. It turns out that the server process is woken up while a `kworker`
kernel thread is already scheduled on that CPU, and the SP5 kernel migrates
the server process to another CPU. The SP6 kernel prefers to delay the
wakeup. However, this situation is too infrequent to explain the difference in
throughput. More importantly, it affects the server process, but throughput is
determined by the client process (which runs at 100% CPU). In short, this is a
red herring.

Second, why is server CPU utilization now so much lower? A quick look at
`mpstat` output suggests it is due to less system time. It can be also
confirmed by running `pidstat -C netserver -u`. Another team member, Gabriel
Krisman Bertazi, who had previously looked into other `netperf` regression,
pointed me at
[commit f35f821935d8](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f35f821935d8df76f9c92e2431a225bdff938169)
(“tcp: defer skb freeing after socket lock is released”).
This led me to a follow-up
[commit 68822bdf76f1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=68822bdf76f10c3dc80609d4e2cdc1e847429086)
(“net: generalize skb freeing deferral to per-cpu lists”).
It moves a call to `sk_defer_free_flush(sk)` from `tcp_v4_rcv()` (running in
the server process context) to `net_rx_action()` (NETRX softirq handler). In
case of the loopback interface, the softirq usually runs on the same CPU which
generated the traffic (here, the client process).

In the test scenario, the upstream change moves even more work from the server
CPU (which has some idle time) to the already fully utilized client CPU,
reducing the rate at which traffic is generated.

To verify this hypothesis, I turned on RPS (Receive Packet Steering) on the
loopback receive queue (this system has 80 CPUs, so the below mask allows
steering to any CPU):

    echo ffff,ffffffff,ffffffff > /sys/class/net/lo/queues/rx-0/rps_cpus

Lo and behold! Throughput with the SP6 kernel is now better than the SP5
reference results by up to **30%.** For the record, merely turning on RPS with
the SP5 kernel already increases the output by up to 20%. That's because the
whole softirq handler runs in parallel with sending further data, so the total
workload is distributed among three CPUs instead of just two.

## Further Directions

Is this the end of my story? Not quite.

First, let's make sure that the regression can be fully explained by moving
work around, which happens to be bad for this specific test case. If the total
amount of work remains unchanged, then running the benchmark on a single CPU
core should yield similar results, right? Let's pin the server and client to
the same CPU. Of course, this reduces the total throughput drastically, but
the results should be equally bad with SP5 and SP6. Except they aren't,
because the scheduler was changed from CFS to EEVDF in v6.6, and the SP6
kernel now makes more frequent context switches, greatly reducing the latency
between `sendto()` in client and `recvfrom()` in server, but at the expense of
total throughput.

Instead, let's try to confirm directly that the difference in throughput can
be fully explained by changes in the NETRX softirq handler. Compare how much
time is spent in `net_rx_action()` with SP5 and SP6 kernels. This excercise is
eye-opening.  Out of the 10s total runtime, more than 3s are spent in the
softirq handler. The CPU time is in fact twice as much, because over 3s are
spent there on the client process CPU and another 3.5s on the server process
CPU. Now compare SP5 and SP6 runs. Softirq time on behalf of the _server_
process does not change (looks like it is even a tiny bit better). However,
softirq time on behalf of the _client_ process increases to almost 5s. What
does that mean? The same amount of data that is sent in 10s with SP5 will need
additional 1.5s with SP6, i.e. 11.5s total. Since the throughput is limited by
the client, this 15% increase in client runtime translates to a 13% reduction
in throughput (1/1.15 ≈ 1-0.13). This can fully explain the regression.

Second, is this microbenchmark even relevant? After all, why would anyone send
heaps of data as UDP or TCP over the loopback interface? If both processes run
on the same host, they'll be better off using Unix-domain sockets, right?
Well, it turns out that similar behaviour is observed on a pair of `veth`
interfaces. These devices are commonly used for container networking. As it
happens, communication between two containers often goes through TCP and/or
UDP, even though they may run on the same host. In short, yes,
[commit 68822bdf76f1](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=68822bdf76f10c3dc80609d4e2cdc1e847429086)
(“net: generalize skb freeing deferral to per-cpu lists”)
may impact some real-world workloads.

Third, how can we deal with the situation? It seems that the upstream change
improves latency for data received over non-local interfaces. It also seems
that this is because the receive and cleanup tasks can be better parallelized.
Local interfaces (loopback or veth) do not benefit from the change, because
packets are in fact received by the sending process. These interfaces would
benefit from parallelizing send and receive. This can be achieved with RPS,
but that's not turned on by default. Either it should be on by default for
local interfaces, or the packet receive should be moved to a worker.

I guess it's time for some public discussion with Eric Dumazet (again).
