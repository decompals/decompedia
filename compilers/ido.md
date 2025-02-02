---
title: IDO
description: 
published: true
date: 2025-02-02T11:20:49.135Z
tags: 
editor: markdown
dateCreated: 2024-11-25T18:43:22.157Z
---

IDO (IRIS Development Option) by SGI was used by developers creating games on SGI hardware, including studios like Rare and DMA.

While multiple versions of IDO exist, only **two** versions appear to have been used for N64 development: **IDO 5.3** and **IDO 7.1**.

## Running IDO
Since IDO is a compiler designed for the IRIX platform, it does not run natively on modern machines. Early decompilation projects, such as *Super Mario 64* and most of *Ocarina of Time*, relied on running IDO through [QEMU](https://www.qemu.org/), an emulation layer. However, this approach had several drawbacks:  

- **Slow execution** due to emulation overhead  
- **Limited platform support**, as it only worked on Linux and macOS  
- **Additional maintenance**, requiring a custom fork of QEMU  

### Recomp
To improve performance and portability, community member **Emil** began working on statically recompiling IDO ("recomp"). This effort produced a significantly faster executable that could be compiled to run on any modern system. Today, this recompiled version is the primary way IDO is used in most decompilation projects.  

The recomp project can be found [here](https://github.com/decompals/ido-static-recomp), with releases available [here](https://github.com/decompals/ido-static-recomp/releases).  

### Decomp
The "holy grail" of reverse engineering a binary is full decompilation, and that applies to IDO as well. A decompilation effort is currently underway. Unlike most other decompilation projects, a significant portion of IDO is written in **Pascal**, alongside C. This makes it one of the first matching decompilation projects for Pascal.  

The matching decompilation project can be found [here](https://github.com/decompals/ido-matching-decomp).

## Projects using this compiler

The following is a list of known projects that use this compiler:

* Animal Forest, mostly IDO 7.1
* Banjo-Kazooie
* Banjo-Tooie
* Chameleon Twist
* Chameleon Twist 2
* Dinosaur Planet
* IDO (IDO Decomp)
* libultra
* Mischief Makers
* Pokémon Snap
* Quest 64
* Space Station Silicon Valley
* Super Mario 64
* The Legend of Zelda: Ocarina of Time, mostly IDO 7.1
* The Legend of Zelda: Majora's Mask, mostly IDO 7.1
* Yoshi's Story
