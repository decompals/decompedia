---
title: Compiler behavior
description: 
published: true
date: 2025-01-21T19:29:35.260Z
tags: 
editor: markdown
dateCreated: 2025-01-21T19:29:35.260Z
---

## Integer and Float Conversions

Float-to-integer conversions use a bespoke instruction `fctiwz`.
```c
s32 float_to_int(f32 a) {
    return a;
}
```
```
fctiwz f0, f1
stwu r1, -0x18(r1)
stfd f0, 0x10(r1)
lwz r3, 0x14(r1)
addi r1, r1, 0x18
blr
```
Integer-to-float conversions have a floating-point subtract (`fsubs`) from a constant 64-bit value that gets loaded into a float register (`lfd`). This value is always `0x4330000080000000` and comes from `.sdata2`.
```c
f32 int_to_float(s32 a) {
    return a;
}
```
```
stwu r1, -0x18(r1)
xoris r0, r3, 0x8000
stw r0, 0x14(r1)
lis r0, 0x4330
lfd f1, "@196"@sda21(r0)
stw r0, 0x10(r1)
lfd f0, 0x10(r1)
fsubs f1, f0, f1
addi r1, r1, 0x18
blr

# .sdata2:0x0 | size: 0x8
.obj "@196", local
	.4byte 0x43300000
	.4byte 0x80000000
.endobj "@196"
```

## Struct copies

The compiler can emit a struct copy in a few ways. If the struct has a size of 64 bytes or less, it generally will copy each value individually:

```c
// sizeof(data) is 16.
typedef struct {
    u32 a[4];
} data;

void test(data *a, data *b) {
    *a = *b;
}
```
```
lwz r5, 0x0(r4)
lwz r0, 0x4(r4)
stw r5, 0x0(r3)
stw r0, 0x4(r3)
lwz r5, 0x8(r4)
lwz r0, 0xc(r4)
stw r5, 0x8(r3)
stw r0, 0xc(r3)
blr
```

It can also choose to use the FPRs to do the copy if the struct has 8-byte alignment, even if there are no floats being used:

```c
// sizeof(data) is 16. Remember that C has to pad x0 to take up 8 bytes
// due to alignment requirements for x8.
typedef struct {
    u8 x0;  
    u64 x8;
} data;

void test(data *a, data *b) {
    *a = *b;
}
```
```
lfd f1, 0x0(r4)
lfd f0, 0x8(r4)
stfd f1, 0x0(r3)
stfd f0, 0x8(r3)
blr
```

If the struct is greater than 64 bytes, the compiler will implicity construct a special kind of for loop:
```c
// sizeof(data) is 100.
typedef struct {
    u32 a[25];
} data;

void test(data *a, data *b) {
    *a = *b;
}
```
```
li r0, 0xc
mtctr r0
subi r5, r3, 0x8
subi r4, r4, 0x8
.L_00000010:
lwzu r3, 0x8(r4)
lwz r0, 0x4(r4)
stwu r3, 0x8(r5)
stw r0, 0x4(r5)
bdnz .L_00000010
lwz r0, 0x8(r4)
stw r0, 0x8(r5)
blr
```
To break it down:
1. The compiler puts `0xC`, or 12, into the counter register with `mtctr`, which is automatically decremented by one every time the `bdnz` jumps back to `.L_00000010`.
2. The loop does two loads and stores per iteration, or in other words, it copies 8 bytes of data per iteration. Since it loops 12 times, this totals to 96 bytes of data.
3. One more load and store is done after the loop (after the counter register is decremented and `bdnz` goes to the next instruction instead of breaking), bringing the total to 100 bytes. 

## Bit fields

C has an odd feature known as *bit fields* which allow the programmer to define variables in a struct that can correspond to data smaller than a byte:

```c
typedef struct {
    u8 b0 : 1;
    u8 b1 : 1;
    u8 b2 : 1;
    u8 b3 : 1;
    u8 b4567 : 4;
} special_flags;

void set_flags(special_flags *flags)
{
    flags->b0 = 1;
    flags->b1 = 0;
    flags->b4567 = 3;
}
```

```
lbz r0, 0x0(r3)
li r4, 0x1
rlwimi r0, r4, 7, 24, 24
stb r0, 0x0(r3)
li r5, 0x0
li r4, 0x3
lbz r0, 0x0(r3)
rlwimi r0, r5, 6, 25, 25
stb r0, 0x0(r3)
lbz r0, 0x0(r3)
rlwimi r0, r4, 0, 28, 31
stb r0, 0x0(r3)
blr
```

This generates the notorious "Rotate Left Word Immediate then Mask Insert M-form" (`rlwimi`) instruction that can be used in a variety of optimizations due to its flexibility, though it's not that hard to follow if you study the entry on page 73 of the [PowerPC User Instruction Set Architecture](https://files.decomp.dev/ppc_isa.pdf):

> *rlwimi RA,RS,SH,MB,ME*
>
>  The contents of register RS are rotated left SH bits. A mask is generated having 1-bits from bit MB + 32 through bit ME + 32 and 0-bits elsewhere. The rotated data are inserted into register RA under control of the generated mask.
> 
> Let RAL represent the low-order 32 bits of register RA, with the bits numbered from 0 through 31.
> 
> rlwimi can be used to insert an n-bit field that is left-justified in the low-order 32 bits of register RS, into RAL starting at bit position b, by setting SH = 32 − b, MB = b, and ME = (b + n) − 1. It can be used to insert an n-bit field that is right-justified in the low-order 32 bits of register RS, into RAL starting at bit position b, by setting SH = 32 − (b + n) , MB = b, and ME = (b + n) − 1.

In the case of the one that's inserting 3 into the four-bit slot `b4567` (`rlwimi r0, r4, 0, 28, 31`), we're inserting n = 4 bits at position b = 28 with the contents of register RS = r4 (which has 0x3 loaded into it) with right-justification, making SH = 32 - (28 + 4) - 1 = 0, MB = 28, and ME = (28 + 4) - 1 = 31.

The decompiler will output the raw functionality of the `rlwimi` without any assumptions about its functionality (you can also generate alternate patterns on [this](https://celestialamber.github.io/rlwinm-clrlwi-decoder/) site), which looks like this:

```c
void set_flags(u8 *arg0) {
    *arg0 |= 0x80;
    *arg0 &= ~0x40;
    *arg0 = (*arg0 & ~0xF) | 3;
}
```

If you see patterns like the above in Melee decomp output, take a look at the struct and you'll likely see a bitfield struct where the write is happening. For instance, `ip->unkDCA = (u8) (ip->unkDCA & ~4);` should be written as `ip->xDC8_word.flags.x5 = 0;`.
