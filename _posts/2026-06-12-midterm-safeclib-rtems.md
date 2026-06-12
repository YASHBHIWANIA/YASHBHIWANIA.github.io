---
layout: post
title: "Midterm Deliverable: Porting Safeclib to RTEMS and Conquering the RSB"
date: 2026-06-12
categories: GSoC RTEMS
---

As part of my Google Summer of Code (GSoC) 2026 journey with the RTEMS Project, my midterm objective was clear: cross-compile the C11 Annex K Bounds Checking Library (`safeclib`) for the RTEMS `sparc/erc32` architecture and seamlessly integrate it into the RTEMS Source Builder (RSB).

What sounded like a straightforward porting task quickly evolved into a deep dive into cross-compilation quirks, bare-metal toolchain configurations, and debugging the build system itself. 

In this post, I will break down exactly how I patched the library for architecture independence, bypassed several RSB extraction quirks, and successfully executed a secure RTEMS application in the SPARC simulator.

### Challenge 1: Architecture Independence (The Patch)
The first major hurdle was the "multilib" problem. The `safeclib` source code had hardcoded checks like `sizeof(uint32_t)` that broke when compiling for different architectures. I created a C patch to replace those hardcoded checks with dynamic GCC macros (like `__SIZEOF_POINTER__`), ensuring the library is truly architecture-independent.

### Challenge 2: Taming the Build System (The RSB Recipe)
Writing the RSB `.cfg` recipe required bypassing several unique build system bugs:
1. **The Git Extraction Bug:** I had to force RSB to actually unpack the code by appending `?checkout=master?pull=1` to the source URL.
2. **The Symlink Bug:** RSB had a hidden bug where it failed to define the source directory. I bypassed this by manually injecting `source_dir_safeclib="rtems-safeclib"` into the `%prep` phase.
3. **The Bare-Metal Compiler Trap:** `safeclib` aggressively tried to enforce Linux-style Stack Smashing Protectors (`ssp.h`) and strict warnings. I bypassed this by injecting custom flags (`-U_FORTIFY_SOURCE -fno-stack-protector -Wno-error`) into the `CFLAGS` during the `./configure` step.

### The Victory: Proof of Execution
After getting the RSB to successfully compile the library, the final test was to link it against a bare-metal RTEMS application.

I wrote a simple "Hello World" RTEMS initialization task that invokes the `strcpy_s` (secure string copy) function from my newly compiled static library. 

Running it in the SPARC Instruction Simulator (`sparc-rtems7-sis`) yielded a perfect execution:

![Safeclib RTEMS Success Output](/assets/images/safeclib-success.png)

With the library fully ported, cross-compiled, and executing securely on the target architecture, the midterm deliverable is officially complete!
