---
layout: post
title: "Kernel Sandbox Mode"
date: "2023-12-04"
tags: [sandbox, kernel]
description:
  I have spent the past couple of months exploring something I call Sandbox
  Mode. It is a way to run Linux kernel code in a constrained execution
  environment. This blog post is an attempt to sort out my thoughts.
---

## Why?

This must always be the first question. Why would anyone want to impose
artificial limitations on their code? The answer is self-protection.

This is nothing new. However, if you go and read the
[Kernel Self-Protection](https://docs.kernel.org/security/self-protection.html)
article, you'll learn that successful self-protection systems are:

* effective,
* on by default,
* require no opt-in by developers,
* have no performance impact,
* do not impede kernel debugging, and
* have tests.

By these measures, my Sandbox Mode idea should be a spectacular failure.

Let's start with the good news. Sandbox Mode is very effective, preventing
even actively malicious kernel code from modifying anything outside a
pre-defined output memory area. If you accept that information leaks are out
of scope, the isolation is just as good as the isolation between kernel mode
and user mode.

There are currently no tests. Writing a test suite requires a lot of effort
which would be wasted if the whole sandbox idea itself is rejected straight
away by the kernel community. Of course, tests will be provided in the initial
submission if there is promising feedback.

Next, Sandbox Mode is on by default. But only if the corresponding config
option is enabled. And that one is much more likely to stay off by default.

From here on, things start to look dire.

Sandbox Mode is definitely opt-in by developers. The core idea is in fact to
make multiple smaller programs from a single bigger one. I wonder if anyone is
able to automate such a task.

Sandbox Mode does have performance impact, although mostly restricted to
entering and exiting sandbox mode. On the positive side, there is at least
zero performance penalty while sandbox code is not running.

Last but not least, the initial implementation makes it harder to debug kernel
code running in a sandbox. In particular, sandbox code cannot be instrumented,
and even in-kernel stack unwinding is broken. OTOH none of these limitations
is inherent to the concept of Sandbox Mode. Debugging of sandbox code can be
enabled with reasonable effort.

**Anyway, with so many drawbacks, why do I even try?**

Although the goal of Sandbox Mode is self-protection, it does not try to
compete with generic mechanisms such as the stack protector or read-only
sensitive variables.

Closest to Sandbox Mode are user mode helpers. If your code is too complex to
run in the kernel, and you consider to write a user space helper, Sandbox Mode
may be a better choice:

* Sandbox Mode *is* (a subset of) kernel mode. It need not be loaded, it does
  not create its own user process and, as a consequence, no other process in
  the system can tamper with it using kill(2), ptrace(2), or any other
  conceivable user-space API.
* Input and/or output need not be serialised and deserialised. All required
  kernel data is made directly accessible to Sandbox Mode, read-only or
  read-write, as appropriate.
* If desired, Sandbox Mode can have access to sensitive kernel data (e.g.
  security or crypto). This is particularly important for kernel lockdown.
* Although not implemented now, it would be trivial to let Sandbox Mode
  disable preemption and/or mask external interrupts. The concept is
  minimalist and flexible: It allows only as little as needed to let your code
  run, but it lets you decide which constraints can be relaxed.

## How?

Sandbox mode runs in non-privileged CPU mode, that is the mode that is
normally used to run user space, not kernel code. It runs within its own
address space, which is a subset of the kernel address space, possibly with
modified permission bits. This allows, for example, to map read-write kernel
data into the sandbox as read-only, so kernel code running in the sandbox can
read but not modify such data.

On CPU interrupt, a trampoline handler enters kernel mode to invoke the
original kernel handler. Note that this is also the only way to preempt code
running in Sandbox Mode.

The page fault CPU exception is intercepted and terminate the sandbox.

## When?

I have written an initial implementation. It seems to work fine. However, the
API to call a function in Sandbox Mode needs some more thought. I have
identified a few potential users and I am now trying to design an easy-to-use
API which could deal with this existing code.
