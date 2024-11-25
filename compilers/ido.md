---
title: IDO
description: 
published: true
date: 2024-11-25T18:43:22.157Z
tags: 
editor: markdown
dateCreated: 2024-11-25T18:43:22.157Z
---

IDO (IRIS Development Option) by SGI was used by developers that developed games on SGI hardware (e.g. Rare, DMA).

Whilst there are many versions of IDO, at the time of writing, only **2** versions appear to have been used for N64 development; IDO 5.3 and IDO 7.1.

## Running IDO
Because IDO is a compiler written for the IRIX platform, it does not run on modern development computers. For early decomp projects like SM64 and most of OOT, IDO was run through [qemu](https://www.qemu.org/), an emulation layer. However, the overhead of qemu was slow, it required development of a separate fork of qemu, and it only worked on Linux and MacOS. 

### Recomp
Later on, community member Emil began working on statically recompiling IDO, which yielded a faster executable that could be run on any modern system. This project eventually became mature enough to the point that it is the main way of running IDO for most projects today.

The recomp project can be found [here](https://github.com/decompals/ido-static-recomp), and releases available [here](https://github.com/decompals/ido-static-recomp/releases).

### Decomp
The "holy grail" of understanding and being able to recompile a binary is to decompile it, and that holds true for IDO. An effort to decompile IDO has begun. A notable difference in this decomp compared to others is that, despite much of the compiler being written in C, there is a fair amount of Pascal as well. This probably is the first matching decomp project for Pascal.

The matching decomp project can be found [here](https://github.com/decompals/ido-matching-decomp).

## Projects using this compiler

The following is a list of known projects that use this compiler:

* Animal Forest, mostly IDO 7.1
* Banjo-Kazooie
* Banjo-Tooie
* Chameleon Twist
* Chameleon Twist 2
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
