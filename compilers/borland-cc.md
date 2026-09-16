---
title: Borland C/C++
description: Turbo C / Borland C++ (16-bit DOS and 32-bit Windows compilers)
published: true
date: 2026-08-20T13:42:59.000Z
tags: compiler, borland, turbo-c, bcc32, bcc
editor: markdown
dateCreated: 2026-08-20T13:42:59.000Z
---

# Borland C/C++

Borland's C/C++ compilers span the DOS and Windows eras, and two distinct lines are relevant to matching decompilation today: the **Turbo C / Turbo C++** line (16-bit DOS compilers, 1987–1992) and **Borland C++ 5.5** (1999, the free 32-bit Windows command-line tools). Both are proprietary and run today through emulation — DOSBox for the 16-bit line, wine (or [wibo](/tools/wibo)) for 32-bit.

## Versions

- **Turbo C 1.0 (1987)** — the first Turbo C; floppy-era 16-bit DOS compiler.
- **Turbo C 2.0 (1988/89)** — the classic DOS-game compiler; e.g. id Software's early titles. C89-strict — rejects `//` comments. archive.org item `turboc20` (assembled from the original floppies).
- **Turbo C++ 3.1 (1992)** — the most-used Turbo release for 16-bit DOS development. archive.org item `turboc3.1_202112`.
- **Borland C++ 5.5 (1999)** — the free command-line tools (`bcc32`), 32-bit Windows; archive.org item `BorlandC55`. Emits OMF objects.

Borland C++ 4.5/5.0 floppies also exist but are not commonly used for matching — see the note under [The 4.5 CD](#the-45-cd).

## Running Borland C/C++

- **Turbo C line** — 16-bit DOS programs; run headless under DOSBox. The [rebrew-toolchains](https://github.com/maci0/rebrew-toolchains) images `rebrew/borland:2.0-win16` / `rebrew/borland:3.1-win16` package `TCC.EXE` with a DOSBox sandbox (sources are staged under a short FAT-safe name and the `.OBJ` is copied back). The objects are Borland 16-bit OMF.
- **bcc32 5.5** — a 32-bit Windows PE; runs under wine via `rebrew/borland:5.5-win32`. Unlike the MSVC line, it does **not** run under [wibo](/tools/wibo): bcc32 calls `GetThreadLocale` from kernel32 at startup, a WinAPI shim wibo does not implement, so wibo aborts with `call reached missing import GetThreadLocale` before any compilation happens. Use wine.

## Identifying a Borland compiled binary

- **Detect It Easy** reports Turbo C 2.0 builds as "Borland C/C++ 1991".
- **Codegen fingerprints** — verified by compiling a probe through TCC 3.1 (`-O1`) and bcc32 5.5 (`-O1`):
  - **Frame pointer always** — every function starts `push ebp; mov ebp,esp` (`55 8b ec`) even at `-O1` (bcc32 probe: 18 frame prologues).  MSVC `/O2` and Watcom omit it — this is the Borland signature.
  - **Real division** — bcc32 emits `mov ecx,3; xor edx,edx; div ecx` for `x/3u`/`x/7u` — no magic-constant multiply.  Its magic set differs from MSVC's too: `0xCCCCCCCD` for `x/10u`, but real `div` for other small divisors.
  - **16-bit line (TCC)** — `push bp; mov bp,sp` framing, `mov bx,N; div bx` (divisor in BX), `fwait` (`9b`) before each x87 op, and `pop bp; ret` epilogues — never `leave` (contrast MSVC 1.5x, which ends functions with `leave`).
- **16-bit NE executables** — Borland segments carry a `[index\x00][name-string][content]` marker at the start of each code segment, and the NE signature sits at `e_lfanew` = 0x40 — versus MSVC's 16-bit linker, which places the NE header at 0x400 and starts segments with code. This cleanly separates Borland from MSVC 16-bit output.
- **Delphi / Pascal string evidence** — Borland RTL strings (length-prefixed Pascal strings) appear in data segments.
- **OMF objects** — Borland's dialect of 16-bit OMF; the compiler version can be extracted from object files (e.g. via `objconv`'s comp.id).

## Known quirks

- **bcc32 has no built-in include/lib path** — the vendored `Include/` and `Lib/` must be passed explicitly (`-I… -L…`).
- **The 4.5 CD** — the Turbo C 4.5 CD (a Windows-IDE-only release) carries **no compiler binary** despite its name: its 207 `.PAK` files are [Quantum archives](https://en.wikipedia.org/wiki/Quantum_compression) (extractable with the rebrew `pak_extract.py`), but no BCC32/BCC/TLINK anywhere. The compiler needs the actual Borland C++ 4.5/5.0 floppy set instead.
- **Turbo C 2.0 determinism** — per-run COMENT ticks in objects are masked in byte-reproducibility gates.

## Where to get it

- archive.org `turboc20` / `turboc3.1_202112` (Turbo C 2.0 / Turbo C++ 3.1 media), `BorlandC55` (official free 5.5 tools).
- The [archaic-toolchains](https://github.com/archaic-toolchains) org mirrors these in its non-MSVC C line, same tree format as its MSVC repos.

## Projects

- [decomp.me](https://decomp.me) — hosts borlandc55 as a win32 compiler.
- [rebrew](https://github.com/maci0/rebrew) — `bcc32` OMF objects are parsed and byte-matched; Turbo C 2.0/3.1 targets compile through the DOSBox images (verified `compile_and_compare` EXACT against a TCC-built object).
- id Software's early DOS titles — the classic Turbo C user.
