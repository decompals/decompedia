---
title: GCC
description: 
published: true
date: 2026-01-27T18:21:28.247Z
tags: compiler
editor: markdown
dateCreated: 2024-12-07T10:04:12.101Z
---

# GCC
GCC is one of the most widely used compilers, and that holds true for game development. A large proportion of N64 games were compiled with GCC. However, there are many different versions of GCC. Due to its open source nature, some development houses made custom modifications to official releases, such as KMC GCC. The community has had to reverse-engineer these differences for some games in order to reproduce a compiler that can be used for matching decomp.

## Versions

Below are the known GCC versions and some examples of games / projects that used them. (todo: add links and pages for these compilers and projects)

### N64
- 2.7.2 (KMC, etc)
- 2.8.1 (Paper Mario)
- SN64
- EGCS sompthin (iQue)

### PSX

PSX games generally use the PSY-Q toolchain, a fork of GCC 2.x. The base GCC  differs between PSY-Q versions, but is generally between 2.6.x and 2.8.x.

### PS2

GCC compilers for the PS2 were forked and supported by SN Systems.

#### Emotion Engine (EE)

- 2.9.x (Fatal Frame, Parappa 2, Twisted Metal Black)
- 2.95.x (Tales of Rebirth)
- 2.96 (Kingdom Hearts)
- 3.2

#### IO Processor (IOP)

- 2.8.x
- 2.95.x

## Patterns

Below is a list of known patterns and optimizations GCC performs on the
previously-mentioned platforms.
**Note** that due to the wide variety of GCC versions and forks, some of these may
not apply to all versions. When possible, patterns will have citations denoting
which game and compiler version they were found in.

### Loads and Stores

#### Load Combining / Coalescing

The following was found in KMC GCC and documented by AngheloAlf [here](https://hackmd.io/eYQR3yfhQymeTGsVM-SpEg).

```c
    // temp_v1 = arg5 - 1;
    // temp_t1 = temp_v1 & (~temp_v1 >> 0x1F);
    temp_t1 = MAX(arg5 - 1, 0);
```

Two adjacent 16-bit loads can be coalesced into a single `lw` when being compared at the same time instead of generating two `lh` instructions. This can only happen if the two are next to each other in memory and the first one is guaranteed to be 4 byte aligned. For example,

```c
struct Test {
    s16 a;
    s16 b;
    s32 for_alignment;
};

int foo(struct Test* t) {
    if (t->a || t->b) {
        bar();
    }
}
```

```
foo:
    addiu $sp, $sp, -0x18
    sw    $ra, 0x10($sp)
    lw    $v0, 0x0($a0) # This is the coalesced loads for `a` and `b`
    beqz  $v0, 1f
      nop
    jal bar
      nop
1:
    lw    $ra, 0x10($sp)
    addiu $sp, $sp, 0x18
    jr    $ra
     nop
```

The compiler can make similar optimizations for combining multiple 8-bit comparisons as well. In the case that the members don't total 4-bytes, it can still coalesce the loads but will mask their combined value when doing the comparison. Non-adjacent members loads can be coalesced as well as long as they're within a 4-byte boundary, and the members between that were skipped will be masked out of the comparison via a combination of lui, ori, and and.

```c
struct Test {
    s8 a;
    s8 b;
    s8 c;
    s8 d;
    s32 for_alignment;
};

int foo(struct Test* t) {
    if (t->a || t->b || t->d) {
        bar();
    }
}
```

```
foo:
    addiu   $sp, $sp, -0x18
    sw      $ra, 0x10($sp)
    lw      $v0, 0x0($a0)
    lui     $v1, 0xffff
    ori     $v1, $v1, 0xff
    and     $v0, $v0, $v1
    beqz    $v0, 1f
      nop
    jal     bar
      nop
1:
    lw    $ra, 0x10($sp)
    jr      $ra
      addiu   $sp, $sp, 0x18
```


Todo add https://github.com/pmret/papermario/wiki/GCC-2.8.1-Tips-and-Tricks

### Shifting

#### Unsigned right shift

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

#### Signed division by 2

The following was found in KMC GCC and documented by AngheloAlf [here](https://hackmd.io/eYQR3yfhQymeTGsVM-SpEg).

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

### Conditionals

#### Conditional range

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

#### SLTI with 0 as immediate

Originally found in SOTN Decomp, exists in exactly one place in the game. Difficult-to-match instruction was 
```
 slti    v0,a0,0 
```
Matching C is 
```
 v0 = ((a0 & (1<<31)) != 0);
```
where a0 is a signed 32-bit integer. Pattern applies to GCC 2.7.2.3 and below.

### Arithmetic

#### Modulo

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

### Loops

#### `-O0` Differences

The following was found in KMC GCC and documented by AngheloAlf [here](https://hackmd.io/eYQR3yfhQymeTGsVM-SpEg).

In `-O0`, `for(;;)` generates different code than `while(1)`.


