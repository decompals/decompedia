---
title: Advance Wars: Dual Strike
description: Matching decompilation of the European Nintendo DS release.
published: true
date: 2026-08-20T00:00:00.000Z
tags: 
editor: markdown
dateCreated: 2026-08-20T00:00:00.000Z
---

# Advance Wars: Dual Strike

A from-scratch matching-decompilation project for the European revision 0 Nintendo DS release, target `AWRP00` (game code `AWRP`). The project starts with ARM9 analysis and requires explicit source coverage for ARM9, all 12 ARM9 overlays, ITCM/DTCM, and ARM7 before completion.

## Links

- [Project repository](https://github.com/ozyz/awds-decomp)
- [Progress on decomp.dev](https://decomp.dev/ozyz/awds-decomp)
- [dsd](https://github.com/AetiasHax/ds-decomp)
- [dsd-ghidra](https://github.com/AetiasHax/dsd-ghidra)
- [objdiff](https://github.com/encounter/objdiff)
- [decomp.me](https://decomp.me/)

## Target and ROM policy

The target is `AWRP00`, Europe revision 0, SHA-1 `03b63875930bd8fdea50902d7d2e0a4614ac36eb`. Contributors verify a legally obtained local ROM using the repository validator. The repository does not distribute the ROM, extracted binaries, filesystem, graphics, audio, text, ARM7 BIOS, proprietary compilers, or other original assets. decomp.me is used only for sanitized single-function ARM9 experiments; local linking and `objdiff` are authoritative.
