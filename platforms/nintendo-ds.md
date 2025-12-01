---
title: Nintendo DS
description: 
published: true
date: 2025-12-01T22:14:30.764Z
tags: 
editor: markdown
dateCreated: 2025-12-01T22:14:30.764Z
---

For years making decompilations of Nintendo DS games was avoided due to the intimidating size of games on the console, lack of documentation surrounding the SDK and the compilers, and not enough tooling being available to set up and contribute to decompilation projects easily.

Today, most decompilations in the DS scene are for pokemon games done by people at pret. However there's been a recent push for more DS decompilations after tools like dsd were developed (based on efforts from GC/Wii) scene to streamline decompilation. 

<h2><a href="/projects/game-boy-advance" style="text-decoration: none; color: darkblue;">Projects</a></h2>

## Resources
- [DS Decompilation Discord](https://discord.gg/gwN6M3HQrA)
- [pret.github.io](https://pret.github.io/)
- [decomp.dev](https://decomp.dev): Project hub and progress info
- [Nintendo DS Architecture: A Practical Analysis by Rodrigo Copetti](https://www.copetti.org/writings/consoles/nintendo-ds/)

## Libraries
- [Nitro SDK](libraries/nitro-sdk)
- [NitroSystem](libraries/nitrosystem)
- [CriWare](libraries/criware)

## Compilers
- [Metrowerks (MWCC)](/compilers/MWCC)

## Tools
- [objdiff](/tools/objdiff)
- [decomp.me](/tools/decomp-me)
- [dsd](https://github.com/AetiasHax/ds-decomp): Toolkit for setting up and maintaining decompilation projects.
- [dsd-ghidra](https://github.com/AetiasHax/dsd-ghidra): Ghidra Extension from synchronizing ghidra's function names with decomp projects made with dsd.
- [ds-rom](https://github.com/AetiasHax/ds-rom): Library/CLI tool for extracting the contents of a DS ROM. Used by dsd.
- [unarm](https://github.com/AetiasHax/unarm): ARM Disassembler library, used by objdiff for DS projects.

### Decompilers
- [m2c](/tools/m2c): m2c has recently been getting support for ARM.
- [Ghidra](/tools/ghidra)