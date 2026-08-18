---
layout: post
title: "Midterm Deliverable: Porting Safeclib to RTEMS, Multi-Arch Recipes, and SPARC Bug Hunting"
date: 2026-06-16
categories: GSoC RTEMS
---

As part of my Google Summer of Code (GSoC) 2026 journey with the RTEMS Project, my midterm objective was clear: cross-compile the C11 Annex K Bounds Checking Library (`safeclib`) for RTEMS, seamlessly integrate it into the RTEMS Source Builder (RSB), and validate it on bare-metal architecture.

What sounded like a straightforward porting task quickly evolved into a deep dive into cross-compilation quirks, dynamic build configurations, and debugging strict-alignment hardware exceptions on the SPARC architecture. 

In this post, I will break down exactly how I patched the library, conquered the RSB, and validated the test suite in the SPARC simulator, documenting every pitfall so future developers can navigate them easily.

### Challenge 1: Architecture Independence (The Patch)
The first major hurdle was the "multilib" problem. The `safeclib` source code relied heavily on hardcoded compiler checks (like `sizeof(uint32_t)`) that inherently break when cross-compiling for different architectures. 

**The Fix:** I created a C patch to rip out those hardcoded checks and replace them with dynamic GCC macros (like `__SIZEOF_POINTER__`). This ensures the library is truly architecture-independent and adapts at compile-time to whatever target RTEMS requires.

### Challenge 2: Taming the Build System (The RSB Recipe)
Writing the RSB `.cfg` recipe required bypassing several unique build system bugs and bare-metal traps:

1. **The Git Extraction Bug:** The RSB initially refused to clone the source properly. **The Fix:** I forced the RSB to unpack the code by explicitly appending `?checkout=master?pull=1` to the source URL.
2. **The Symlink Bug:** RSB had a hidden bug where it failed to define the source directory, breaking the build pipeline. **The Fix:** I bypassed this by manually injecting `source_dir_safeclib="rtems-safeclib"` into the `%prep` phase.
3. **The Bare-Metal Compiler Trap:** `safeclib` aggressively tried to enforce Linux-style Stack Smashing Protectors (`ssp.h`) and strict GCC warnings. **The Fix:** I injected custom bare-metal override flags (`-U_FORTIFY_SOURCE -fno-stack-protector -Wno-error`) into the `CFLAGS` during the `./configure` step.

### Challenge 3: Dynamic Multi-Architecture Support & Linker Errors
To make the recipe robust for the RTEMS ecosystem, it needed to support cross-architecture builds out of the box (MR !269). 

1. **Dynamic `pkg-config`:** I replaced the hardcoded `rtems-bsps` placeholder with dynamic host/bsp extraction (`%{_host}-$(basename %{with_rtems_bsp})`) during the `%build` phase. This allows `pkg-config` to correctly extract compilation flags for *any* target architecture (SPARC, RISC-V, etc.).
2. **Bypassing the Syslog Linker Error:** On bare-metal targets, linking the performance tests failed entirely due to an undefined reference to `syslog`—a utility that simply doesn't exist on these lightweight embedded targets. **The Fix:** I used `sed` to strip the `tests` directory from the generated Makefile targets. This prevented the broken test suite from interrupting the build while ensuring the core library, headers, and `.pc` files installed correctly.

### The First Milestone: Minimal Proof of Execution
With the library successfully building, the initial test was to link it against a basic bare-metal RTEMS application to verify the hooks. 

I wrote a minimal RTEMS initialization task that invokes the `strcpy_s` (secure string copy) function from my newly compiled static library. Running it in the SPARC Instruction Simulator (`sparc-rtems7-sis`) yielded a clean execution:

![Safeclib RTEMS Minimal Success](/assets/images/safeclib-hello-success.png)

### The Final Boss: Full Test Suite Validation
With basic execution proven, the final step was running the entire `safeclib` test suite on the simulator. This is where the real bare-metal debugging began.

Initially, the test suite threw catastrophic OS-level crashes. I had to debug and apply two critical OS-level fixes to my RTEMS initialization task:
1. **FPU Exceptions (Code 38):** The simulator was crashing due to illegal floating-point unit usage. **The Fix:** I injected the `RTEMS_FLOATING_POINT` attribute into the Init task.
2. **File Descriptor Exhaustion:** The library's `fopen` tests were crashing the OS because RTEMS defaults to a highly restrictive file descriptor limit. **The Fix:** I bumped `CONFIGURE_MAXIMUM_FILE_DESCRIPTORS` up to 32.

### The Result: 116 PASS and Hardware Bug Hunting
Once the OS configurations were patched, the test suite results immediately jumped to **116 PASS**. 

![Safeclib RTEMS Full Test Suite Results](/assets/images/safeclib-suite-results.png)

More importantly, clearing away the OS crashes isolated the remaining 15 Fails and 6 Timeouts (in functions like `test_memcpy32_s` and `test_mbstowcs_s`). As suspected, these are not OS errors, but genuine SPARC strict-alignment data access exceptions (Trap 9s) and infinite loops. 

Standard x86 desktop processors often hide these memory alignment flaws, proving exactly why validating this library on strict bare-metal architectures like SPARC is so critical. I am currently documenting these isolated hardware traps to file as upstream bug reports to the `safeclib` maintainers.

With the library ported, dynamically compiling across architectures in the RSB, and the SPARC validation complete, my midterm deliverables are officially wrapped. Next up: building the `riscv-rtems7-gcc` toolchain and seeing how RISC-V handles these exact same memory alignment tests!
