---
layout: post
title: "Cross-Compiling C Libraries for Bare-Metal: Escaping the Autotools Trap"
date: 2026-05-29 16:35:00 +0530
categories: [GSoC, RTEMS, C]
tags: [cross-compilation, embedded, autotools]
---

# Cross-Compiling C Libraries for Bare-Metal: Escaping the Autotools "Ignore" Trap

**By Yash Bhiwania** | GSoC 2026 Contributor for RTEMS

When porting a standard C library to a bare-metal embedded real-time operating system (RTOS) like RTEMS, you quickly learn that standard build tools make a lot of assumptions about your environment. Specifically, tools like Autoconf expect a standard POSIX-compliant OS to be sitting underneath them. 

For my Google Summer of Code project, I am porting the `safeclib` library to RTEMS. Recently, I ran into a classic cross-compilation trap involving autoconf probes, linker errors, and the temptation to just "brute-force" the build. 

Here is how I fell into the trap, and more importantly, the architectural pivot required to do it the "RTEMS way."

### The "Blindfold" Method: Ignoring Unresolved Symbols
Initially, to get the autotools `./configure` script to pass its generic C compiler dummy test without failing on missing RTEMS scaffolding, I bypassed the linker errors using a brute-force flag:

LDFLAGS="-Wl,--unresolved-symbols=ignore-all"

It worked... sort of. The build moved forward, but as my mentor pointed out, ignoring undefined symbols on an autoconf probe is a terrible idea. It effectively puts a blindfold on the compiler. It allows the autoconf probes to falsely report features as "present" (like `fork()` or `getpid()`) even when they are entirely missing from the bare-metal environment. This guarantees silent crashes down the line. 

### The Pivot: Linking the Board Support Package (BSP)
To fix this, I dropped the `ignore-all` flag and decided to point the configuration script directly to the installed SPARC RTEMS7 BSP (`erc32`). This forces the script to look at the *actual* symbols available in the embedded environment.

export RTEMS_ROOT=$HOME/development/rtems/7/sparc-rtems7/erc32

./configure \
  --host=sparc-rtems7 \
  --prefix=$RTEMS_ROOT \
  --enable-static \
  --disable-shared \
  CC="sparc-rtems7-gcc -qrtems -B$RTEMS_ROOT/lib/" \
  CPPFLAGS="-I$RTEMS_ROOT/include"

### Unveiling the True Linker Errors
Once the blindfold was off, the script failed immediately on the very first probe: `checking whether the C compiler works... no`. 

Digging into the `config.log`, I found the exact culprit. The RTEMS compiler was refusing to link the autoconf `conftest.c` dummy file, throwing these undefined references:

* `_ISR_Stack_size`
* `_ISR_Stack_area_begin`
* `_Scheduler_Table`
* `_Thread_Information`

**The Aha Moment:** These are not standard C library functions; they are RTEMS kernel configuration symbols! In RTEMS, an application requires a configuration block (typically via `<rtems/confdefs.h>`) to allocate memory for the scheduler, threads, and interrupts. Because the autotools dummy script is just an empty `int main()` without this header, the RTEMS linker panicked.

### The Path Forward
In cross-compilation, there are generally two professional ways to handle autoconf probes failing on bare-metal:
1.  **Use a Dummy Config Library:** RTEMS provides `librtemsdefaultconfig.a`, which defines stubs for these exact configuration macros to satisfy the linker during tests.
2.  **Hard-code the Variables:** Bypass the executable tests entirely and manually inject a `config.site` file that tells the build system exactly which OS features are (and aren't) available. 

I am currently discussing the best architectural approach with my mentors to integrate this cleanly into the RTEMS Source Builder (RSB). 

*If you want to follow along with the code, I am pushing all my Work-In-Progress (including some MSVC test stub adjustments) to my personal RTEMS GitLab fork here:* [https://gitlab.rtems.org/Yash/rtems-safeclib](https://gitlab.rtems.org/Yash/rtems-safeclib)
