---
title: ProDG
description: 
published: true
date: 2026-11-05T04:43:27.969Z
tags: 
editor: markdown
dateCreated: 2026-11-05T04:43:27.969Z
---

# Header
ProDG is a commercial compiler by SN Systems as part of their ProDG line of development tools. The GameCube target (and earlier Nintendo console targets) are based on GCC 2.95.x. ProDG is a proprietary compiler, unlike GCC, which was licensed out to developers for use on the Nintendo GameCube (and Game Boy Advance).

Identifying a ProDG compiled binary
The SDK build stamps
Dolphin SDK libraries compiled with ProDG include version strings such as:

<< Dolphin SDK - GX release build: Feb  7 2003 04:01:13 (0x2301) >>
These can be used to identify the exact SDK revision, which helps date the game and bracket the compiler version.

__FILE__ strings
ProDG's GCC emits __FILE__ strings for assertion diagnostics. On GameCube these often survive as absolute Windows-style paths:

D:/NGC/<project>/prog/main.c
These give the game's original source-file names and the developer's project codename, which are strong evidence for translation-unit boundaries.

Register allocation signature
ProDG's GCC 2.95 allocates GPRs in the order r0, r9, r11, r10, r8, r7, r6, r5, r4, r3 before callee-saved registers, and FPRs in the order f0, f13, f12, f11, f10, f9, f8, ... f1. A function whose first live pointer lands in r9 and whose loop counter lands in r10/r7 is characteristic.

Matching notes
ProDG GameCube code is GCC 2.95.x. Register assignment and instruction scheduling follow the stock compiler's internal decisions, which are documented in the compiler source:

gcc/config/rs6000/rs6000.h — REG_ALLOC_ORDER
gcc/local-alloc.c — the pseudo processing priority QTY_CMP_PRI = floor_log2(n_refs) * n_refs * size / (death - birth), ties broken by creation order
gcc/config/rs6000/rs6000.md — floatsidf2 (signed int-to-double, xoris + a 2^52 + 2^31 pool constant) vs floatunssidf2 (unsigned, plain 2^52 pool constant)
Practical consequences:

The first GPR pseudo the allocator processes lands in r9; "wrong register" near-misses are usually the allocator processing two pseudos in a different order, which is driven by statement order (creation order), reference counts and lifetimes — not by declaration order alone.
A 2^52 + 2^31 double in the original's .rodata next to an int-to-float conversion means the source cast was signed; 2^52 means unsigned. This can be read directly off the original binary.
Common source-shape fixes for scheduler/allocator walls: an empty asm to obscure a value's range and restore a redundant extension (__asm__("" : "=r"(x) : "0"(x));), a clobber plus a bound local plus a pure barrier to restore a skipped argument move, hoisting a pointer into a local with a barrier to fix call-argument order, and writing if/else with direct stores in each arm instead of a hoisted initializer to fix sunk constants.
When a whole region scores near zero on tiny functions, suspect the compiler or flags before the source: game code may be ProDG while the SDK and middleware are CodeWarrior, and the two compilers need different tooling and flags.
Decompilation Projects using this compiler
GameCube
Resident Evil 4 (debug build) — game code ProDG 3.9.3, SDK/middleware CodeWarrior
Need for Speed: Most Wanted / Underground — ProDG GC 3.9.3
The Sims 2 — ProDG (GCC 2.95.3, SN BUILD v1.76)
Known ProDG games without a decompilation project
Viewtiful Joe (GVJE08)
P.N.03
Many EA and Capcom titles of the era