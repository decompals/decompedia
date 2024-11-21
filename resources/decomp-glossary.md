---
title: Decomp Glossary
description: 
published: true
date: 2024-11-21T21:15:13.441Z
tags: 
editor: markdown
dateCreated: 2024-11-19T17:56:47.312Z
---

# Decomp Glossary
This glossary is here to give you a general gist of patterns you can expect to see when reverse-engineering an optimizing compiler. They're compiled with the compiler and settings that the Melee project uses, which, relevant for this discussion, include:
1. MWCC 1.2.5n 
2. `-O4,p`, highest-level optimization 
3. `-enum int`, making enums always take up four bytes of memory

The following typedefs are used:

```c
/// A null pointer
#define NULL ((void*) 0)
/// A signed 8-bit integer
typedef signed char s8;
/// A signed 16-bit integer
typedef signed short s16;
/// A signed 32-bit integer
typedef signed long s32;
/// A signed 64-bit integer
typedef signed long long s64;
/// An unsigned 8-bit integer
typedef unsigned char u8;
/// An unsigned 16-bit integer
typedef unsigned short u16;
/// An unsigned 32-bit integer
typedef unsigned long u32;
/// An unsigned 64-bit integer
typedef unsigned long long u64;
```
> Note: The explanations provided for each sample are *generalizations* of the compiler's behavior, they likely won't be true for 100% of all possible code permutations. An important skill to have when working on a matching decomp is to never *assume* that something that affects codegen can be ruled out.
{.is-info}

## Compiler behavior

### Loading from a pointer

### Integer and Float Conversions

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

### Struct copies

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

### Bit fields

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

The decompiler will output the raw functionality of the `rlwimi` without any assumptions about its function (you can also generate alternate patterns on [this](https://celestialamber.github.io/rlwinm-clrlwi-decoder/) site), which looks like this:

```c
void set_flags(u8 *arg0) {
    *arg0 |= 0x80;
    *arg0 &= ~0x40;
    *arg0 = (*arg0 & ~0xF) | 3;
}
```

If you see patterns like the above in Melee decomp output, take a look at the struct and you'll likely see a bitfield struct where the write is happening. For instance, `ip->unkDCA = (u8) (ip->unkDCA & ~4);` should be written as `ip->xDC8_word.flags.x5 = 0;`.


## Melee-specific behavior

### GObjs and GET_ inlines

The majority of data in Melee is grouped into kinds of "objects" of the name `HSD_?Obj`, with ? corresponding to a letter. You can see all the types of objects [here](https://github.com/doldecomp/melee?tab=readme-ov-file#sysdolphinbaselib). By far the most common of these is the global/game object, or `GObj`, which is a kind of "generic" object that contains a pointer called `user_data` at offset `0x2C`. This pointer `user_data` points to many kinds of important data types in the game including fighters, items, stages, and menu elements, and because of this, it's strongly suspected that the developers used some type of getter inline to simplify the process of casting this `user_data` from a `void *` to the type. So instead of this:
```c
void test(HSD_GObj *arg3, HSD_GObj *arg4) {
    Item *temp_r5 = (Item *)arg3->user_data;
    Fighter *temp_r6 = (Fighter *)arg4->user_data;
}
```
you should write this:
```c
void test(HSD_GObj *arg3) {
    Item *temp_r4 = GET_ITEM(arg3);
    Fighter *temp_r6 = GET_FIGHTER(arg4);
}
```
We also currently follow a naming convention for objects and the structs inside `GObj` from debug strings in the codebase that reveal them to us. For instance, `"fp->ground_or_air == GA_Ground"` is a real string written by the devs found inside of `ft/ftlib.c`. There's no set-in-stone rules you have to follow, but in this case, it'd be preferrable to write this:
```c
// If fighter_gobj were the only gobj, you could just write it as gobj instead.
void test(HSD_GObj *gobj, HSD_GObj *fighter_gobj) {
    Item *ip = GET_ITEM(gobj);
    Fighter *fp = GET_FIGHTER(fighter_gobj);
}
```
There's also an important cosmetic change to how GObjs are written that we've made because of how it can represent multiple kinds of data. We have a `typedef` for each of the struct types that `user_data` can hold which replaces `HSD_` with the struct name, making it look like this:
```c
void test(Item_GObj *gobj, Fighter_GObj *fighter_gobj) {
    Item *ip = GET_ITEM(gobj);
    Fighter *fp = GET_FIGHTER(fighter_gobj);
}
```
There's primarily two reasons for structuring GObjs like this:
1. It gives context to the decomper as to what object a given function takes without having to look at the implementation.
2. It allows us to influence the decompiler and have it write the types inside `user_data` more efficiently. 

### JObjs

### State tables for items and stages

Most functions in the object files of the `it` and `gr` folders consist of what are referred to as *callback functions*, whose address gets saved to a function pointer that gets called somewhere in the code. Each item and stage contains structs that contain these pointers: `ItemStateTable` for items, and `StageCallbacks` and `StageData` for stages. The decompiler isn't able to extract these in most cases, so you need to open the assembly yourself to be able to include them. Here's what the state table for item `itheiho` looks like:


```c
ItemStateTable it_803F83F0[] = { { -1, it_802D88CC, it_802D88D4, it_802D8910 },
                                 { 0, it_802D8984, it_802D8A54, it_802D8CC8 },
                                 { -1, it_802D8DB4, it_802D8DBC, it_802D8E44 },
                                 { -1, it_802D8E4C, it_802D8E54, it_802D8EA4 },
                                 { 2, it_802D9274, it_802D9384,
                                   it_802D95F4 } };

```

```
# .data:0x0 | 0x803F83F0 | size: 0x50
.obj it_803F83F0, global
	.4byte 0xFFFFFFFF
	.4byte it_802D88CC
	.4byte it_802D88D4
	.4byte it_802D8910
	.4byte 0x00000000
	.4byte it_802D8984
	.4byte it_802D8A54
	.4byte it_802D8CC8
	.4byte 0xFFFFFFFF
	.4byte it_802D8DB4
	.4byte it_802D8DBC
	.4byte it_802D8E44
	.4byte 0xFFFFFFFF
	.4byte it_802D8E4C
	.4byte it_802D8E54
	.4byte it_802D8EA4
	.4byte 0x00000002
	.4byte it_802D9274
	.4byte it_802D9384
	.4byte it_802D95F4
.endobj it_803F83F0
```

```
(in objdiff)
.data
00000000: ff ff ff ff 00 00 00 00    00 00 00 00 00 00 00 00 
00000010: 00 00 00 00 00 00 00 00    00 00 00 00 00 00 00 00 
00000020: ff ff ff ff 00 00 00 00    00 00 00 00 00 00 00 00 
00000030: 00 00 00 02 00 00 00 00    00 00 00 00 00 00 00 00 
```

You'll notice if you look at the bytes of this struct in the `.data` tab of objdiff that the places where the function pointers should be are zeroed out, which is because the linker still has to be run to finalize where the addresses will be, as [explained](decomp-intro-melee#compiling-and-linking) in the intro section. The assembly is getting the labels not from the empty slots here, but a specific table in the object file that isn't carried over into the assembly:

```
(in the objdump output of itheiho.o)
RELOCATION RECORDS FOR [.data]:
OFFSET   TYPE              VALUE
00000004 UNKNOWN           it_802D88CC
00000008 UNKNOWN           it_802D88D4
0000000c UNKNOWN           it_802D8910
00000014 UNKNOWN           it_802D8984
00000018 UNKNOWN           it_802D8A54
0000001c UNKNOWN           it_802D8CC8
00000024 UNKNOWN           it_802D8DB4
00000028 UNKNOWN           it_802D8DBC
0000002c UNKNOWN           it_802D8E44
00000034 UNKNOWN           it_802D8E4C
00000038 UNKNOWN           it_802D8E54
0000003c UNKNOWN           it_802D8EA4
00000044 UNKNOWN           it_802D9274
00000048 UNKNOWN           it_802D9384
0000004c UNKNOWN           it_802D95F4
```

## Misc behavior

### Declaration before initialization rule

Melee is compiled with ANSI C, the first C specification by ANSI, and its largest oddity that can catch you off guard if you don't know about it is the fact that all variables *must* be declared at the start of the scope before any other types of statements can be made. This is why you may recieve this strange error:

```
### mwcceppc.exe Compiler:

User break, cancelled...
#    File: src\melee\it\items\itnokonoko.c
# ----------------------------------------
#      21: HSD_JObj* jobj = HSD_GObjGetHSDObj(gobj);
#   Error: ^^^^^^^^
#   expression syntax error
#   Too many errors printed, aborting program
ninja: build stopped: subcommand failed.
```

from this code:

```c
void ansi(Item_GObj *gobj) {
    Item *ip = GET_ITEM(gobj);
    ip->xD5C = 0;

    HSD_JObj* jobj = HSD_GObjGetHSDObj(gobj);
    HSD_JObjAddRotationY(jobj, 0.15707964f);
}
```

To fix it, you would move the `jobj` declaration above the assignment to `ip->xD5C`:

```c
void ansi(Item_GObj *gobj) {
    Item *ip = GET_ITEM(gobj);
    HSD_JObj* jobj = HSD_GObjGetHSDObj(gobj);

    ip->xD5C = 0;
    HSD_JObjAddRotationY(jobj, 0.15707964f);
}
```

You could also encase the `jobj` declaration in its own scope if the function is excessively long:

```c
void ansi(Item_GObj *gobj) {
    Item *ip = GET_ITEM(gobj);
    ip->xD5C = 0;
    
    // a bunch of stuff

    {
        HSD_JObj* jobj = HSD_GObjGetHSDObj(gobj);
        HSD_JObjAddRotationY(jobj, 0.15707964f);
    }
}
```

### Assumed integer return on undeclared functions

MWCC by default will allow you generate an object file that calls functions you don't have a prototype for, even if the call sites contradict themselves:
```c
// fdsa and fdsa_2 are not defined anywhere in the TU, but both of these compile

void asdf(s32 a) {
    fdsa(a);
}

void asdf_2(s32 a, f32 b) {
    fdsa_2(a);
    fdsa_2(b);
}
```

This isn't a big deal normally because the linker will see this and throw a `function has no prototype` error, but since in a decomp we aren't running the linker often, strange errors like this can pop up:

```
### mwcceppc.exe Compiler:

User break, cancelled...
#    File: src\melee\it\items\itnokonoko.c
# ----------------------------------------
#      18: Item *ip = GET_ITEM(gobj);
#   Error:                          ^
#   cannot convert
#   'int' to
#   'struct Item *'
```

This arises because the compiler will assume that all undeclared functions have an integer return, which contradicts with the `Item` type we're trying to assign the `GET_ITEM` macro to, which it's interpreting as a function since we haven't written the macro definition (`#include "it/inlines.h"`). In the Melee repo, passing `--require-protos` will pass a flag to the compiler to throw the proper warning:

```
### mwcceppc.exe Compiler:

User break, cancelled...
#    File: src\melee\it\items\itnokonoko.c
# ----------------------------------------
#      19: Item *ip = GET_ITEM(gobj);
#   Error:            ^^^^^^^^
#   function has no prototype
#   Too many errors printed, aborting program
```
