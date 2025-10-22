---
layout: post
title: "Linux Command Line Parser"
date: "2025-10-22"
tags: [kernel]
description:

  After improving the handling of the 'tsx' kernel parameter, I feel like I
  keep rediscovering the same things I already found out. Let me dump some
  facts here to help me (and possibly others) when needed again.
---

There should be a long introductory part explaining all the logical
events leading up to the current sorry state. Sorry, too little time
for that. Maybe over a beer at a conference…

## Common Parts

As of 2025, all kernel parameters should be eventually parsed with
`parse_args()`. This is a slightly overengineered mechanism to split
the command line into individual key-value pairs while allowing to
quote white space.

Individual options are described by `struct kernel_param` (see
`<linux/moduleparam.h>`). This structure is best suited for module
parameters, which may be also given on the command line (prefixed with
the module name and a dot), they usually appear in sysfs, and
sometimes can even be writable.

## Normal Parameters

Normal parameters passed to the kernel are processed relatively late:
in `kernel_init()`, which runs in PID 1 context (before first exit to
user mode). They are defined by a *`level`*`_param_cb()` macro, where
*level* is the name of an initcall level, namely:

1. core
2. postcore
3. arch
4. subsys
5. fs
6. device
7. late

The parameters for each level are parsed just before calling the
corresponding initcalls. The idea is to associate parameter parsing
with initcalls. For example, parameters for a `subsys_initcall()`
function should be defined using `subsys_param_cb()`.

## Early Parameters

Beware! Besides a regular `core_param_cb(name, ops, arg, perm)`
(parsed just before core initcalls), there is also an **unrelated**
`core_param(name, var, type, perm)`. The latter is used for real core
parameters (e.g. `panic_on_warn`), which are parsed *much* earlier
than normal parameters. More specifically, they are handled by the
`parse_args()` call in `start_kernel()`, after the kernel prints
“Kernel command line: ” to the message log and before it lists unknown
parameters.

Historically, such parameters were defined with a `__setup()` macro.
Their definition is now stored in a `struct obs_kernel_param` (IIUC
`obs` stands for “obsolete”), and they are processed just after core
parameters, see `obsolete_checksetup()`. The setup function takes a
single string argument, which will point to the equals sign (`=`)
separating the parameter name from its value, *unless* that character
was appended to the option name given to `__setup()`. The equals sign
must not be included in the name if the option takes no value, but
then you must check whether the value starts with an equals sign (and
skip it if necessary).

The `__setup()` macro is defined using the `__setup_param()` macro
(which is also used directly in a few places). The last parameter of
the `__setup_param()` is called `early` and it is used to initialise a
`struct obs_kernel_param` field of the same name. It is set to zero by
`__setup()` and all direct users, but it is set to one by another
macro, called `early_param()`. This macro is used for “true” early
parameters, processed by `parse_early_param()`. Note that they also
use `struct obs_kernel_param` but differently than the obsolete
`__setup()`-style parameters. The name should never include the equals
sign, and the setup function receives only the actual value (or `NULL`
if none).

The exact point in the boot process when `parse_early_param()` is
called depends on the target architecture.

As of 6.17, most architectures call it from `setup_arch()`:
- arc
- arm
- arm64
- csky
- hexagon
- loongarch
- mips
- m68k
- riscv
- sh
- sparc
- s390
- xtensa
- x86

A few call it extremely early (even before `start_kernel()`):
- microblaze
- nios2
- powerpc

The remaining architectures rely on a fallback call in
`start_kernel()`, just before processing parameters defined by
`core_param()`:
- alpha
- openrisc
- parisc
- um
