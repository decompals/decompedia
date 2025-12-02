---
title: Nintendo 3DS
description: 
published: true
date: 2025-12-02T01:04:24.067Z
tags: 
editor: markdown
dateCreated: 2025-12-02T01:04:24.067Z
---

The 3DS decompilation scene is still in it's infancy. There's been little research into compiler codegen patterns and current tooling for decompilation projects is little more than python scripts shared between projects.

Most 3DS games are written in C++. Some are written in C (especially ports of other games written in C, or indie games). Unity games released for 3ds are written in C# then compiled to C++ using IL2CPP. The size of the code for these games can vary a lot (anywhere from 484KB for Ikachan to 13MB+ for Super smash bros 3DS). 

Instead of overlays, 3DS games use "CROs" which are relocatable module files similar to Dll files on windows. Sometimes these CROs come with names for exported functions, acting as a nice source of symbols.

One other interesting thing is that 3DS games are likely compiled with `--split-sections` as a default flag, which turns each function into it's own TU. And the linker rearranges these TUs, sorting them alphabetically or putting ones calling each other together as a primitive optimization. 
For games that have partial symbols, the alphabetical sorting done by the linker can help people working on said games come up with names for unknown functions.

<h2><a href="/projects/game-boy-advance" style="text-decoration: none; color: darkblue;">Projects</a></h2>

## Resources
- [Nintendo 3DS Architecture: A Practical Analysis by Rodrigo Copetti](https://www.copetti.org/writings/consoles/nintendo-3ds/)

## Libraries
- [CTR SDK](libraries/ctr-sdk)
- [NintendoWare for CTR](libraries/nw4c)
- [PIA](libraries/pia)
- [NEX](libraries/nex)
- [CTR Face Library](libraries/ctr-face-library)
- [Mobiclip library](libraries/mobiclip-3ds)
- [SEAD](libraries/sead)

## Compilers
- [ARM C/C++ Compiler (ARMCC)](/compilers/ARMCC)

## Tools
- [decomp.me](/tools/decomp-me)
- [ninfs](https://github.com/ihaveamac/ninfs): Python tool to view the contents of 3DS ROMs.