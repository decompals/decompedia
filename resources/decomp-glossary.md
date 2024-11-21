---
title: Decomp Glossary
description: 
published: true
date: 2024-11-21T17:23:34.939Z
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
