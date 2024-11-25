---
title: Space Station Silicon Valley
description: 
published: true
date: 2024-11-25T18:48:35.187Z
tags: 
editor: markdown
dateCreated: 2024-11-25T18:48:35.187Z
---

[Space Station Silicon Valley](https://en.wikipedia.org/wiki/Space_Station_Silicon_Valley) (SSSV) is an N64 game released in October 1998. There is an active decompilation in progress at https://github.com/mkst/sssv.

### General Info

* 8MB cartridge. 
* F3DEX microcode
* Libultra 2.0I
* Uses RNC compression for a number of assets

### Versions
The game was released in US and Europe regions and there are two retail versions: version 1.0 and 1.1. 

There are a couple of unreleased versions.

### Code

#### Overlays
There are 2 code overlays.

##### Overlay 1
This overlay contains the code for the intro cutscene as well as the language select menu that is displayed when running the EU version.

In the 1.1 ROM this is compiled without optimisation and is therefore significantly easier to match than the 1.0 ROM.

##### Overlay 2
This contains "everything else"; both the game as well as the main menu.

### ROM Hacks
There are a couple of ROM hacks available for SSSV. Written by ozidual they address the 2 main bugs that plague the retail releases:

* 1.0 - game crashes when Expansion Pak is present
* 1.0+1.1 - Fat Bear Mountain trophy has no collision and is therefore not collectable

Entrypoint is patched to check for the presence of the Expansion Pak:
```as
 glabel func_80125900
 /* 1000 80125900 3C0A8000 */  lui        $t2, %hi(D_80000319)
 /* 1004 80125904 814A0319 */  lb         $t2, %lo(D_80000319)($t2)
 /* 1008 80125908 294A0006 */  slti       $t2, $t2, 0x6
 /* 100C 8012590C 15400002 */  bnez       $t2, .L80125918
 /* 1010 80125910 3C09006A */   lui       $t1, (0x6A1E30 >> 16)
 /* 1014 80125914 3C09002A */  lui        $t1, (0x2A1E30 >> 16)
 .L80125918:
 /* 1018 80125918 3C088016 */  lui        $t0, %hi(D_8015E1D0)
 /* 101C 8012591C 35291E30 */  ori        $t1, $t1, (0x2A1E30 & 0xFFFF)
 /* 1020 80125920 2508E1D0 */  addiu      $t0, $t0, %lo(D_8015E1D0)
 .L80125924:
 /* 1024 80125924 2129FFF8 */  addi       $t1, $t1, -0x8
 /* 1028 80125928 AD000000 */  sw         $zero, 0x0($t0)
 /* 102C 8012592C AD000004 */  sw         $zero, 0x4($t0)
 /* 1030 80125930 1520FFFC */  bnez       $t1, .L80125924
 /* 1034 80125934 21080008 */   addi      $t0, $t0, 0x8
 /* 1038 80125938 3C0A8012 */  lui        $t2, %hi(func_80125950)
 /* 103C 8012593C 3C1D8016 */  lui        $sp, %hi(sc)
 /* 1040 80125940 254A5950 */  addiu      $t2, $t2, %lo(func_80125950)
 /* 1044 80125944 01400008 */  jr         $t2
 /* 1048 80125948 27BD03D0 */   addiu     $sp, $sp, %lo(sc)
 /* 104C 8012594C 00000000 */  nop
```

## Evo's Space Adventures
SSSV was ported to PlayStation and shares a lot of code and resources from original. There is a [decompilation project](https://github.com/mkst/esa) which was created to demonstrate that PSX decompilation is possible by leveraging the existing tools created as part of N64 decomp projects.

### External Resources

* https://tcrf.net/Space_Station_Silicon_Valley_(Nintendo_64)
