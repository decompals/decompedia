---
title: ProDG
description: 
published: true
date: 2026-11-05T05:04:26.395Z
tags: 
editor: markdown
dateCreated: 2026-11-05T04:43:27.969Z
---

# SN Systems ProDG on GameCube

ProDG is a commercial compiler from SN Systems and part of its ProDG development
tools. The GameCube target, like earlier Nintendo console targets, is based on GCC
2.95.x. Unlike GCC itself, ProDG is proprietary software that was licensed to
developers for Nintendo GameCube and Game Boy Advance development.

## Identifying a ProDG-compiled binary

Several clues can help identify a binary built with ProDG. Each is useful evidence;
no single clue is conclusive on its own.

### Dolphin SDK build stamps

Dolphin SDK libraries often contain version strings such as:

```text
<< Dolphin SDK - GX release build: Feb  7 2003 04:01:13 (0x2301) >>
```

These stamps can identify the exact SDK revision. That helps date the game and
narrow down the compiler version in use.

### `__FILE__` strings

ProDG's GCC emits `__FILE__` strings for assertion diagnostics. On GameCube, these
often survive as absolute Windows-style paths:

```text
D:/NGC/<project>/prog/main.c
```

The paths can reveal original source filenames and the developer's project codename.
They are also useful evidence when inferring translation-unit boundaries.

### Register-allocation signature

GCC 2.95's ProDG build allocates general-purpose registers (GPRs) in this order
before moving to callee-saved registers:

```text
r0, r9, r11, r10, r8, r7, r6, r5, r4, r3
```

Floating-point registers (FPRs) are allocated in this order:

```text
f0, f13, f12, f11, f10, f9, f8, ... f1
```

A function where its first live pointer lands in `r9` and its loop counter lands in
`r10` or `r7` is characteristic of this allocation order.

## Matching ProDG output

ProDG's GameCube compiler is based on GCC 2.95.x. Register assignment and
instruction scheduling follow the compiler's internal decisions. Relevant details
are documented in the GCC source:

- [`gcc/config/rs6000/rs6000.h`](https://gcc.gnu.org/git/?p=gcc.git;a=blob;f=gcc/config/rs6000/rs6000.h;hb=refs/heads/releases/gcc-2.95): `REG_ALLOC_ORDER`.
- [`gcc/local-alloc.c`](https://gcc.gnu.org/git/?p=gcc.git;a=blob;f=gcc/local-alloc.c;hb=refs/heads/releases/gcc-2.95): pseudo processing priority, `QTY_CMP_PRI = floor_log2(n_refs) * n_refs * size / (death - birth)`, with ties broken by creation order.
- [`gcc/config/rs6000/rs6000.md`](https://gcc.gnu.org/git/?p=gcc.git;a=blob;f=gcc/config/rs6000/rs6000.md;hb=refs/heads/releases/gcc-2.95): `floatsidf2` (signed integer-to-double conversion, using `xoris` and a `2^52 + 2^31` pool constant) versus `floatunssidf2` (unsigned conversion, using a plain `2^52` pool constant).

### Practical implications

- The first GPR pseudo processed by the allocator lands in `r9`. A near-match with a
  value in the wrong register often means the compiler processed two pseudos in a
  different order. That order depends on statement order, reference counts, and
  lifetimes, rather than declaration order alone.
- A `2^52 + 2^31` double in the original `.rodata` beside an integer-to-float
  conversion indicates a signed source cast. A `2^52` constant indicates an
  unsigned cast. The constant can be read directly from the original binary.
- Common source-shape adjustments for scheduler or allocator mismatches include:
  - An empty assembly statement to hide a value's range and restore a redundant
    extension: `__asm__("" : "=r"(x) : "0"(x));`
  - A clobber, a register-bound local, and a pure barrier to restore a skipped
    argument move.
  - Hoisting a pointer into a local with a barrier to change call-argument order.
  - Writing `if`/`else` with direct stores in each arm instead of a hoisted
    initializer to change where constants are scheduled.
- If many tiny functions score near zero, check the compiler and flags before
  changing the source. Game code may use ProDG while SDK and middleware code use
  CodeWarrior; those compilers need different tools and flags.

These are matching techniques, not proof of the original source spelling. Verify
each one against the target binary.

## Decompilation projects using ProDG

### GameCube

- **Resident Evil 4 (debug build)** — game code uses ProDG 3.9.3; SDK and middleware
  use CodeWarrior.
- **Need for Speed: Most Wanted / Underground** — ProDG GC 3.9.3.
- **The Sims 2** — ProDG (GCC 2.95.3, SN BUILD v1.76).

## Known ProDG games without a decompilation project

- Viewtiful Joe (`GVJE08`)
- P.N.03
- Many Electronic Arts and Capcom titles from the same era