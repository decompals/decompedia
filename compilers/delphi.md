---
title: Delphi
description: Borland Delphi 1.0 (16-bit Windows Pascal compiler)
published: true
date: 2026-08-20T13:42:59.000Z
tags: compiler, delphi, borland, pascal
editor: markdown
dateCreated: 2026-08-20T13:42:59.000Z
---

# Delphi

Delphi is Borland's RAD (rapid application development) Pascal product. **Delphi 1.0 (1995)** targets 16-bit Windows 3.x and is the version relevant to decompilation: it compiles to genuine NE 6.01 executables via its command-line compiler `DCC.EXE`. The Borland Pascal dialect and the 16-bit NE output make it a distinct, self-contained corner of the DOS/Windows toolchain family.

## Running Delphi

`DCC.EXE` is a 16-bit DOS program, so it runs headless under DOSBox. The [rebrew-toolchains](https://github.com/maci0/rebrew-toolchains) image `rebrew/delphi:1.0-win16` packages the exact command-line toolchain: `DCC.EXE` (Delphi Compiler 8.0, Sep 1995), `DELPHI.DSL` (compiler symbol table), the `CMDLINE.PAK` tools, and the RTL/VCL units (`UNITS.PAK` + `LIB.PAK`). The sources stage under a short FAT-safe name and the produced `.EXE` is copied back.

The `.PAK` files are **Quantum archives**; the rebrew tree ships a reverse-engineered extractor (`pak_extract.py`) and documents the recipe.

### Matching caveat

Delphi's Borland ABI has **no matchable compiler profile** — the Delphi codegen cannot be reproduced from C source for byte matching, so Delphi functions are documented as blockers in tooling. The toolchain remains useful for verification-style research (compile + NE parse of the output).

## Identifying a Delphi compiled binary

- **16-bit NE executables** — Borland segments carry a `[index\x00][name-string][content]` marker at the start of each code segment, with the NE signature at `e_lfanew` = 0x40 (MSVC's 16-bit linker puts it at 0x400).
- **Pascal strings** — Delphi data segments are full of length-prefixed (Pascal) strings rather than NUL-terminated ones; string scanning tools recognize them.
- **RTL evidence** — Delphi VCL/runtime strings; `diec` reports the Delphi version.

## Where to get it

- archive.org item `delphi10` — the Delphi 1.0 media (RTL/VCL units have no public tarball, so toolchains commit the verified tree instead).
- The [archaic-toolchains](https://github.com/archaic-toolchains) org mirrors it in its non-MSVC line.

## Projects

- [rebrew](https://github.com/maci0/rebrew) — uses the Delphi 1.0 toolchain to compile and parse NE targets (validated against `holiday.exe`, a Delphi 1.0 VCL app).
