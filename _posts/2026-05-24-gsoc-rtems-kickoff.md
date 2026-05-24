---
layout: post
title:  "GSoC 2026: Bringing Safe C to Space with RTEMS"
date:   2026-05-24 18:00:00 +0530
categories: gsoc rtems
---

Hello world! I am Yash, a 3rd-year B.Tech student in Communication and Computer Engineering. This summer, I am incredibly excited to announce that I will be participating in Google Summer of Code 2026 with the **RTEMS Project**. 

For those who might not know, RTEMS is a real-time operating system that flies on ESA and NASA spacecraft, runs medical devices, and powers automotive controllers. 

### The Problem
RTEMS currently uses Newlib as its C library. However, Newlib does not implement the standard C11 Annex K bounds-checking interfaces. This means RTEMS developers writing code for safety-critical systems have no standardized way to use bounds-checked memory and string functions (like `strcpy_s` or `memset_s`). They usually have to rely on the unsafe originals or write their own wrappers.

### My Project
My mission for the summer is to package `safeclib`—a library that implements all 45 Annex K functions—through the RTEMS Source Builder (RSB). By the end of August, developers will be able to easily build and link these safe functions for any RTEMS Board Support Package (BSP) just by adding `-lsafec`. 

The pre-coding community bonding period is officially wrapping up today. I've spent the last couple of weeks finalizing the RSB recipe design with my mentors, Joel Sherrill and Gedare Bloom, and verifying upstream compatibility.

Coding officially begins tomorrow! My immediate next step is to draft the RSB `.cfg` and `.bset` skeleton and get the RTEMS `Init` task scaffolding running under QEMU SPARC to test the library on bare metal.

I'll be posting weekly technical updates right here. Stay tuned!
