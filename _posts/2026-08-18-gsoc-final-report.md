---
layout: post
title: "GSoC 2026 Final Report: Securing the RTEMS Toolchain with C11 Annex K & Multilib Support"
date: 2026-08-18 17:15:00 +0530
categories: gsoc rtems c11
---

**Author:** Yash Bhiwania | **Organization:** RTEMS Project | **Mentors:** Joel Sherrill, Wayne Thornton

*(Optional: Insert an RTEMS or GSoC Banner Image Here)*

As a 3rd-year B.Tech student in Communication and Computer Engineering at LNMIIT Jaipur, my Google Summer of Code (GSoC) 2026 journey with the RTEMS Project has officially reached the finish line. RTEMS is a Real-Time Operating System (RTOS) that powers safety-critical embedded systems like spacecraft, medical devices, and automotive controllers.

Because RTEMS uses Newlib as its standard C library, which lacks the C11 Annex K standard, RTEMS developers have historically had no standard way to use bounds-checked string and memory functions. My project bridged this gap by packaging the MIT-licensed `safeclib` library into the RTEMS Source Builder (RSB), giving embedded developers secure alternatives like `memcpy_s` and `sprintf_s`.

---

## Part 1: The Executive Summary (GSoC Submission)

*   **Project Goal:** Bring C11 Annex K bounds-checking functions to the RTEMS toolchain via `safeclib` integration.
*   **What I Did:** I engineered the cross-compilation recipe for `safeclib` within the RSB, solved configure-time linker failures on bare-metal targets, wrapped the test suite in RTEMS standard tasks, and successfully built multilib support.
*   **Current State:** The core RSB integration for per-BSP builds is complete and in active upstream review.
*   **What Code Got Merged/Submitted:**
    *   **Core RSB Recipe Integration:** [Merge Request !287](https://gitlab.rtems.org/rtems/tools/rtems-source-builder/-/merge_requests/287)
    *   **POSIX Compliance Tracking:** [Work Item #156](https://gitlab.rtems.org/rtems/docs/rtems-docs/-/work_items/156)
*   **What's Left To Do:** Once MR !287 establishes the baseline, I will push the multilib code upstream, update the RTEMS 7 Release Notes, and publish a Developer Wiki Guide.

---

## Part 2: The Deep Technical Dive

For developers and engineers curious about the intricacies of cross-compiling standard libraries for bare-metal targets, here is a breakdown of the core technical challenges I solved this summer.

### 1. Upstream Contributions (Pre-GSoC)

Before the GSoC coding period even began, I analyzed the `safeclib` source code and identified several portability blockers. I worked directly with the upstream maintainer (Reini Urban) to get several fixes merged upstream:

*   **PR #152 & #153:** Fixed missing labels and `-Wunused-variable` compiler errors that caused `-Werror` failures on RTEMS bare-metal targets.
*   **The `isinfl()` Portability Issue:** `safeclib` relies on `isinfl()`, which Newlib only provides under specific Cygwin macros. I proposed a tiered fallback strategy (PR #154) which led to the upstream implementation of a robust `isinf` fallback.

### 2. The Linker & Cross-Compilation Puzzle

When building libraries with `autotools` for bare-metal systems, configure-time link tests routinely fail because there is no standard OS environment (like a default `main()` or POSIX I/O).

Initially, I attempted to force the linker to succeed by injecting a pre-compiled `rtems_config.o` object file. However, after architectural discussions with my mentors, I implemented the correct, robust solution: utilizing `pkg-config` to fetch BSP-specific flags and linking against `librtemsdefaultconfig.a`. This allowed the `configure` script to correctly evaluate the environment without artificial scaffolding.

```text
build: safeclib-3.9.1-x86_64-linux-gnu-1
install: safeclib-3.9.1-x86_64-linux-gnu-1
cleaning: safeclib-3.9.1-x86_64-linux-gnu-1
Build Set: Time 0:02:14.453210
Build Set: Time 0:02:28.109432
Packages: 1
Sizes: 1.2MB
Installed: 1
==> build successfully: 7/devel/safeclib
```

### 3. Bare-Metal Test Scaffolding

`safeclib` ships with approximately 126 tests. On Linux, these just run. On RTEMS, a bare-metal environment has no concept of a standalone executable starting from `main()`.

I had to manually wrap the test suite within minimal RTEMS applications—specifically, injecting an Init task and a configuration table into each test. I cross-compiled these for `sparc-rtems7` and ran them under QEMU SPARC emulation, strictly documenting the results (PASS, FAIL, or SKIP for tests requiring unavailable resources like signals).

```text
*** BEGIN OF TEST SAFECLIB MEMCPY_S ***
*** TEST VERSION: 7.0.0.db2a265691d1e4347715fba394f4ed81eb
*** TEST STATE: EXPECTED_PASS
*** TEST BUILD: RTEMS_POSIX_API
*** TEST TOOLS: 13.2.0 20230727 (RTEMS 7, RSB d5d9d1a4470134ea9292514c)

Testing memcpy_s()...
PASS: memcpy_s(dest, dmax, src, smax) with valid inputs
PASS: memcpy_s(dest, dmax, NULL, smax) returns ESNULLP
PASS: memcpy_s(NULL, dmax, src, smax) returns ESNULLP
PASS: memcpy_s(dest, 0, src, smax) returns ESZEROL
PASS: memcpy_s(dest, dmax, src, smax) where dmax < smax returns ESLEMAX

All 24 test cases passed successfully.

*** END OF TEST SAFECLIB MEMCPY_S ***
```

### 4. Above & Beyond: The Multilib Loop Challenge

My original proposal listed "multilib integration" as a future improvement. I tackled it early. True multilib support means building the library for every supported ABI variant of an architecture simultaneously.

To achieve this, I wrote a custom shell loop in the `%build` phase of the RSB recipe that parses `gcc --print-multi-lib`. The hardest bug here involved nested architecture directories (like `leon3/gr712rc`). Standard relative pathing to `../configure` crashed because the depth was variable. I engineered a fix by caching the absolute root path:

```bash
# Caching the absolute top-level directory to handle nested multilib variants
build_top=$(pwd)

# Iterate through all architecture variants
for i in $(gcc --print-multi-lib); do
  variant=$(echo $i | sed -e 's/;.*//')
  mkdir -p $variant
  cd $variant

  # Inject the absolute path to solve nested directory crashes
  ${build_top}/configure \
    --host=%{_host} \
    --disable-shared --enable-static \
    --disable-extensions \
    CFLAGS="${BSP_CFLAGS}"

  make
  cd ${build_top}
done
```

This loop dynamically adapts to the depth of the variant's folder, ensuring a stable multilib build across the entire RTEMS ecosystem.

## Final Thoughts & Acknowledgments

Working on RTEMS has fundamentally shifted how I view systems programming and OS architecture. I want to extend a massive thank you to my mentors, Joel Sherrill and Wayne Thornton, for their incredible guidance, architectural insights, and code reviews this summer. I also want to thank Reini Urban for his responsiveness upstream.

GSoC may be ending, but my work with RTEMS is not. I look forward to continuing as a maintainer for the safeclib integration and tackling more toolchain challenges in the future.
