---
title: Decomp Glossary
description: 
published: true
date: 2025-01-21T19:34:24.873Z
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

see [here](/resources/glossary/compiler-behavior)

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

### JObjs and its inlines

The inlines that you will likely see the most of are the getters and setters for joint objects, or `JObj`s, which contain a lot of leftover debug assertions. For example, this:

```c
// in src/sysdolphin/baselib/jobj.h

static inline void HSD_JObjGetScale(HSD_JObj* jobj, Vec3* scale)
{
    HSD_ASSERT(823, jobj);
    HSD_ASSERT(824, scale);
    *scale = jobj->scale;
}

static inline void HSD_JObjSetScale(HSD_JObj* jobj, Vec3* scale)
{
    HSD_ASSERT(760, jobj);
    HSD_ASSERT(761, scale);
    jobj->scale = *scale;
    if (!(jobj->flags & JOBJ_MTX_INDEP_SRT)) {
        HSD_JObjSetMtxDirty(jobj);
    }
}

```

```c
void jobj_demo(Item_GObj *gobj) {
    HSD_JObj *jobj = HSD_GObjGetHSDObj(gobj);
    Vec3 scale;
    HSD_JObjGetScale(jobj, &scale);
    scale.x += 0.05f;
    HSD_JObjSetScale(jobj, &scale);
}
```

has this decompiler output:

```c
static s8 @193[8] = { 0x6A, 0x6F, 0x62, 0x6A, 0x2E, 0x68, 0, 0 };
static s8 @194[5] = { 0x6A, 0x6F, 0x62, 0x6A, 0 };

void jobj_demo(void *arg0) {
    f32 sp14;
    f32 sp10;
    f32 spC;
    HSD_JObj *temp_r31;
    s32 temp_cr0_eq;
    s32 var_r3;
    u32 temp_r4;

    temp_r31 = arg0->unk28;
    if (temp_r31 == NULL) {
        __assert(@193, 0x337U, @194);
    }
    spC = temp_r31->scale.x;
    sp10 = temp_r31->scale.y;
    sp14 = temp_r31->scale.z;
    spC += 0.05f;
    if (temp_r31 == NULL) {
        __assert(@193, 0x2F8U, @194);
    }
    temp_r31->scale.x = spC;
    temp_r31->scale.y = sp10;
    temp_r31->scale.z = sp14;
    if (!(temp_r31->flags & 0x02000000)) {
        temp_cr0_eq = temp_r31 == NULL;
        if (temp_cr0_eq == 0) {
            if (temp_cr0_eq != 0) {
                __assert(@193, 0x234U, @194);
            }
            temp_r4 = temp_r31->flags;
            var_r3 = 0;
            if (!(temp_r4 & 0x800000) && (temp_r4 & 0x40)) {
                var_r3 = 1;
            }
            if (var_r3 == 0) {
                HSD_JObjSetMtxDirtySub(temp_r31);
            }
        }
    }
}
```

You can see the split between the get and set call in `spC += 0.05f;`. The decompiler is able to successfully guess that `temp_r31` is a jobj at least, since it knows `HSD_JObjSetMtxDirtySub()` takes one. The other most obvious sign you have a jobj inline here is the several `__assert` functions generated from the `HSD_ASSERT()` macro; `@193` and `@194` are strings that say `jobj.h` and `jobj`. 

Luckily the fact that `__assert` has a line number in the second argument means we can find the correct inline 100% of the time without having to even analyze what the rest of the code is doing: `0x337U` is 823, and `0x2F8U` is 760, unique indices present only in the `HSD_JObjGetScale` and `HSD_JObjSetScale` functions respectively. However if you look closely you'll notice there seemingly is a missing assert corresponding to the `scale` variable for both the getter and setter; this is because it got optimized out since `scale` was declared on the stack and thus its address can never be `NULL`, which is what the assert is checking for. Theoretically this could also happen to the `gobj` as well, but it is incredibly rare. 

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

You'll notice if you look at the bytes of this struct in the `.data` tab of objdiff that the places where the function pointers should be are zeroed out, which is because the linker still has to be run to finalize where the addresses will be, as [explained](/resources/decomp-intro-melee#compiling-and-linking) in the intro section. The assembly is getting the labels not from the empty slots here, but a specific table in the object file that isn't carried over into the assembly:

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
