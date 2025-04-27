---
title: Assembly Patterns
description: 
published: true
date: 2025-04-27T20:56:35.390Z
tags: 
editor: markdown
dateCreated: 2025-04-27T20:47:35.791Z
---

# Header
As C and C++ are structured languages, the assembly they get compiled to will exhibit some common patterns. Furthermore, certain operations will be consistently optimised in a certain way, giving rise to more patterns.

## Divide and multiply by 2

Assembly:
> andi    v0,a1,0xfffe

C code:
> unsigned short my_array[9];
>
> my_array[i / 2]

To index an array of 2-byte values (here, unsigned shorts), the index is multiplied by two. The code logic happens to require that the index be divided by two, first. The optimisation here does that in one step by ANDing with 0xFFFE (-2).

## Range check

Assembly:
> lhu     v1,0x10(sp)
> nop
> addiu   v0,v1,-0xe9
> sltiu   v0,v0,0xd
> bnez    v0,144

C code:
> if (wrk > 0xE8 && wrk < 0xF6)

One might be tempted to translate this as `if (wrk - 0xE9 < 0xD)`, but this is actually an optimised check to see if the variable is between 0xE8 and 0xF6. 0xE9 is one lower than the lower bound, and 0xF6 - 0xE8 = 0xE, which is one more than the range.