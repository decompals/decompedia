---
title: GCC
description: 
published: true
date: 2024-12-07T10:04:15.056Z
tags: compiler
editor: markdown
dateCreated: 2024-12-07T10:04:12.101Z
---

# GCC
GCC is one of the most widely used compilers, and that holds true for game development. A large proportion of N64 games were compiled with GCC. However, there are many different versions of GCC. Due to its open source nature, some development houses made custom modifications to official releases, such as KMC GCC. The community has had to reverse-engineer these differences for some games in order to reproduce a compiler that can be used for matching decomp.

Below are the known GCC versions and some examples of games / projects that used them. (todo: add links and pages for these compilers and projects)

### N64
- 2.7.2 (KMC, etc)
- 2.8.1 (Paper Mario)
- SN64
- EGCS sompthin (iQue)
### PSX
- PSY-Q toolchain

## Decompilation patterns
Todo add https://github.com/pmret/papermario/wiki/GCC-2.8.1-Tips-and-Tricks

Todo add https://hackmd.io/eYQR3yfhQymeTGsVM-SpEg

##### Unsigned right shift

Used to perform logical shifts to the left for 16-bit and 8-bit values.
```
 (var_a0 << 0x10) >> 0x13
```
The processor is ensuring var_a0 is a u16 by removing the highest 16 bits. then it is shifting it back to the right by 0x10 plus 3. This can be simplified as the following:
```
 (u16)var_a0 >> 0x13
```
If var_a0 is already a u16, then you can just write the following:
```
 var_a0 >> 3
```
If var_a0 would have been an u8, the pattern would have looked like the following:
```
 (var_a0 << 0x18) >> 0x1B
```

##### Signed division by 2

There are three different patterns involving a signed variable divided by 2. The variant depends from the size of the type.
```
  s32 temp_v0;
  // temp_v0 = ((s32)(temp_v0 + ((u32)temp_v0 >> 0x1F)) >> 1)
  temp_v0 /= 2;
```
  
```
  s16 temp_v0;
  // temp_v0 = ((s32)(temp_v0 + (((u32)(temp_v0 << 0x10)) >> 0x1F))) >> 1
  temp_v0 /= 2;
```
  
```
  s8 temp_v0;
  // temp_v0 = ((s32)(temp_v0 + (((u32)(temp_v0 << 0x18)) >> 0x1F))) >> 1
  temp_v0 /= 2;
```

##### Conditional range

Used when comparing a variable to a specific range of integers.
```
 if (((u32)(var_a0 - 0xE)) < 9) 
```
This can be translated into
```
 if (var_a0 >= 0xE && var_a0 < 9 + 0xE) 
```
And can be simplified as 
```
 if (var_a0 > 13 && var_a0 < 23) 
```
The way it works is that if var_a0 is less than 14, when 14 is subtracted and then converted to unsigned then it will be greater than 9, effectively being equivalent to the conditional range prior to optimization.

##### Modulo

Used when performing a modulo to a power of 2
```
 s32 a = i + n;
 s32 b = a;
 if (b < 0) {
     b = i + n + 15;
 }
 x = a - (b >> 4) * 0x10;
```
Can also be written as
```
 x = (i + n) % 16
```
##### SLTI with 0 as immediate

Originally found in SOTN Decomp, exists in exactly one place in the game. Difficult-to-match instruction was 
```
 slti    v0,a0,0 
```
Matching C is 
```
 v0 = ((a0 & (1<<31)) != 0);
```
where a0 is a signed 32-bit integer. Pattern applies to GCC 2.7.2.3 and below.