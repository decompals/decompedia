---
title: GCC
description: 
published: true
date: 2026-02-08T19:39:49.889Z
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

### Arithmetic

#### Modulo by power of 2

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

#### Signed division by 2

The following was found in KMC GCC and documented by AngheloAlf [here](https://hackmd.io/eYQR3yfhQymeTGsVM-SpEg).

There are three different patterns involving a signed variable divided by 2. The variant depends from the size of the type.

```c
  s32 temp_v0;
  // temp_v0 = ((s32)(temp_v0 + ((u32)temp_v0 >> 0x1F)) >> 1)
  temp_v0 /= 2;
```
  
```c
  s16 temp_v0;
  // temp_v0 = ((s32)(temp_v0 + (((u32)(temp_v0 << 0x10)) >> 0x1F))) >> 1
  temp_v0 /= 2;
```
  
```c
  s8 temp_v0;
  // temp_v0 = ((s32)(temp_v0 + (((u32)(temp_v0 << 0x18)) >> 0x1F))) >> 1
  temp_v0 /= 2;
```

#### Division by constant using multiply+shift (`MULT_HI`)

The following was originally taken from [Paper Mario](https://github.com/pmret/papermario/wiki/GCC-2.8.1-Tips-and-Tricks), which uses GCC 2.8.1.

GCC will sometimes optimize a floating-point division with a constant into
a multiply-and-shift operation, taking advantage of support for
64-bit multiply results.

```c
f32 x;
// y is some constant value
x / y;
```

becomes

```c
f32 x;
// x = MULT_HI(x, constant) >> shift
x = (float)(((u64)x * constant >> 0x20) >> shift);
```

The following list of constants/shifts were taken from [this blog](https://flaviojslab.blogspot.com/2008/02/integer-division.html).

|Multiply|Shift|Original|
|--------|-----|--------|
|0x069C16BD|	0|	665/25756|
|0x10624DD3|	4|	1/250|
|0x10624DD3|	5|	1/500|
|0x10624DD3|	6|	1/1000|
|0x2AAAAAAB|	0|	1/6|
|0x30C30C31|	3|	1/42|
|0x38E38E39|	1|	1/9|
|0x4EC4EC4F|	3|	1/26|
|0x51EB851F|	5|	1/100|
|0x55555556|	0|	1/3|
|0x66666667|	1|	1/5|
|0x66666667|	2|	1/10|
|0x66666667|	3|	1/20|
|0x66666667|	4|	1/40|
|0x66666667|	5|	1/80|
|0x66666667|	6|	1/160|
|0x66666667|	7|	1/320|
|0x66666667|	8|	1/640|
|0x6BCA1AF3|	5|	1/76|
|0x88888889|	8|	1/480|
|0x92492493|	3|	1/14|
|0xA0A0A0A1|	7|	1/204|
|0xAAAAAAAB|	1|	1/3|
|0xAAAAAAAB|	2|	1/6|
|0xAAAAAAAB|	3|	1/12|
|0xAAAAAAAB|	4|	1/24|
|0xAE147AE1|	5|	17/800|
|0xB21642C9|	5|	1/46|
|0xB60B60B7|	5|	1/45|
|0xBA2E8BA3|	3|	1/11|
|0xEA0EA0EB|	0|	32/35|

### Shifting

#### Unsigned right shift (Signed/Unsigned conversions)

Used to perform logical shifts to the left for 16-bit and 8-bit values.
```c
 (var_a0 << 0x10) >> 0x13
```
The processor is ensuring var_a0 is a u16 by removing the highest 16 bits. then it is shifting it back to the right by 0x10 plus 3. This can be simplified as the following:
```c
 (u16)var_a0 >> 0x13
```
If var_a0 is already a u16, then you can just write the following:
```c
 var_a0 >> 3
```
If var_a0 would have been an u8, the pattern would have looked like the following:
```c
 (var_a0 << 0x18) >> 0x1B
```

### Conditionals

#### Conditional range

Used when comparing a variable to a specific range of integers.
```c
 if (((u32)(var_a0 - 0xE)) < 9) 
```
This can be translated into
```c
 if (var_a0 >= 0xE && var_a0 < 9 + 0xE) 
```
And can be simplified as 
```c
 if (var_a0 > 13 && var_a0 < 23) 
```
The way it works is that if var_a0 is less than 14, when 14 is subtracted and then converted to unsigned then it will be greater than 9, effectively being equivalent to the conditional range prior to optimization.

#### SLTI with 0 as immediate

Originally found in SOTN Decomp, exists in exactly one place in the game. Difficult-to-match instruction was 
```
 slti    v0,a0,0 
```
Matching C is 
```c
 v0 = ((a0 & (1<<31)) != 0);
```
where a0 is a signed 32-bit integer. Pattern applies to GCC 2.7.2.3 and below.

### Branching

#### `likely` Instructions (`bnel`, `bnezl`, etc.)

The following was originally taken from [Paper Mario](https://github.com/pmret/papermario/wiki/GCC-2.8.1-Tips-and-Tricks), but the behavior has been seen
in GCC 2.9 on PS2 as well (Twisted Metal Black).

If you encounter likely instructions, these mean that the delay slot instruction is only executed if the branch is taken. Sometimes your code is equalvent but you still get these instructions (or don't get them when you want them). Try inverting the condition, as this sometimes coerces the compiler into using a `likely` instruction.

### Loops

#### `-O0` Differences

The following was found in KMC GCC and documented by AngheloAlf [here](https://hackmd.io/eYQR3yfhQymeTGsVM-SpEg).

In `-O0`, `for(;;)` generates different code than `while(1)`.

#### Negative struct offsets in loops

The following was originally documented for [Paper Mario](https://github.com/pmret/papermario/wiki/GCC-2.8.1-Tips-and-Tricks), which uses GCC 2.8.1.

You may see the asm do this:

```c
void fx_73_update(EffectInstance* arg0) {
    SomeStruct* structTemp;
    s32 i;

    structTemp = arg0->data;

    if (arg0->numParts > 1) {
        structPlus20 = temp_a1 + 0x20;
        do {
            if (structPlus20->unk0 <= 0) {
                structPlus20->unk-1C--;
                if (structPlus20->unk-1C >= 0xA) {
                    structPlus20->unk0 = -1;
                }
            }
            i++;
            structPlus20 += 0x24;
        } while (i < arg0->numParts);
    }
}
```

Note the negative offsets (`unk-1C`, for example) and how structPlus20 is `0x20` bytes into the struct. You can calculate the correct offsets by taking the `0x20` and subtracting `0x1C` to get `structTemp->unk_04`. However, to get the code to actually generate these negative offsets, you need to increment the struct temp pointer as well as the normal loop iterator: `for (i = 1; i < numParts; i++, structTemp++) {`

### Switch Statements / Jump Tables

#### "Irregular" Switches

The following was originally documented for [Paper Mario](https://github.com/pmret/papermario/wiki/GCC-2.8.1-Tips-and-Tricks), which uses GCC 2.8.1.

"Irregular" switches (as deemed by [m2c](github.com/matt-kempster/m2c)) are switches that are small enough that they do not need a jumptable, and may appear as an if/else chain in m2c's output.

The following is an example of what you might see from m2c for an irregular switch:

```c
if (var != 1) {
  if (var < 4) {
    // case 0, 2, 3
    temp = 2;
  } else {
    // default case
    temp = 5;
  }
} else {
  // case 1
  temp = 10;
}
```

which can be matched with the following switch statement:

```c
switch (var) {
  case 1:
    temp = 10;
    break;
  case 0:
  case 2:
  case 3:
    temp = 2;
    break;
  default:
    temp = 5;
    break;
}
```

##### Irregular Switch Irregularities

Irregular switches may include a seemingly unused register checking for a certain condition; this may imply that the original code contained a case that was identical to the default case but was explicitly provided e.g. `slti t6, s5, 2`

the `t6` register (result of the comparison) will never be used in the following switch, but the comparison that sets it will appear in the asm:

```c
switch (temp_t5) {
  case 0:
  case 1:
    /* matches default case! */
    var1 = 10;
    var2 = 20;
    break;
  case 8:
    var1 = 15;
    var2 = 40;
    break;
  case 11:
    var1 = 30;
    var2 = 5;
    break;
  default:
    var1 = 10;
    var2 = 20;
    break;
}
```

### Register Allocation (Regalloc)

#### Branch-Invariant Code

The following was found in GCC 2.9 991111 (PS2) in Twisted Metal: Black.

Some branch folding optimizations can influence GCC's register allocation.
Take `mathfDist2()`:

```c++
float mathfDist2(FVECTOR* origin, FVECTOR* dst)
{
    float dx, dy;
    if (dst == NULL) {
        dx = origin->x;
        dy = origin->y;
    } else {
        dx = (dst->x - origin->x);
        dy = (dst->y - origin->y);
    }
    return sqrtf(dx * dx + dy * dy);
}
```

A diff shows that we likely have the right number of temporary variables,
since no extra registers are allocated. However, registers are being
swapped unexpectedly:

```
TARGET                                                   CURRENT (80)
db9d8:    addiu   sp,sp,-0x10                            db9d8:    addiu   sp,sp,-0x10
db9dc:    bnez    a1,db9f0 ~>                            db9dc:    bnez    a1,db9f0 ~>
db9e0:    sd      ra,0(sp)                               db9e0:    sd      ra,0(sp)
db9e4:    lwc1    $f1,4(a0)                       r      db9e4:    lwc1    $f3,4(a0)
db9e8:    b       dba08 ~>                               db9e8:    b       dba08 ~>
db9ec:    lwc1    $f0,0(a0)                              db9ec:    lwc1    $f0,0(a0)
db9f0: ~> lwc1    $f1,4(a1)                       r      db9f0: ~> lwc1    $f3,4(a1)
db9f4:    lwc1    $f3,4(a0)                       r      db9f4:    lwc1    $f1,4(a0)
db9f8:    lwc1    $f0,0(a1)                       r      db9f8:    lwc1    $f2,0(a1)
db9fc:    lwc1    $f2,0(a0)                       r      db9fc:    lwc1    $f0,0(a0)
dba00:    sub.s   $f1,$f1,$f3                     r      dba00:    sub.s   $f3,$f3,$f1
dba04:    sub.s   $f0,$f0,$f2                     r      dba04:    sub.s   $f0,$f2,$f0
dba08: ~> mul.s   $f1,$f1,$f1                     r      dba08: ~> mul.s   $f1,$f0,$f0
dba0c:    mul.s   $f0,$f0,$f0                     r      dba0c:    mul.s   $f0,$f3,$f3
dba10:    add.s   $f12,$f0,$f1                    r      dba10:    add.s   $f12,$f1,$f0
```

It's possible that the two branches originally duplicated some code
(the `sqrtf()` call, in this case). GCC would have pulled the code out and
executed it after the branch.

```c++
float mathfDist2(FVECTOR* origin, FVECTOR* dst)
{
    float result;
    if (dst == NULL) {
        float dx = origin->x;
        float dy = origin->y;
        result = sqrtf(dx * dx + dy * dy);
    } else {
        float dx = (dst->x - origin->x);
        float dy = (dst->y - origin->y);
        result = sqrtf(dx * dx + dy * dy);
    }
    return result;
}
```

This code matches exactly to the original once compiled, though it is quite ugly.
We can simplify it:

```c++
float mathfDist2(FVECTOR* origin, FVECTOR* dst)
{
    if (dst == NULL) {
        return sqrtf(origin->x * origin->x + origin->y * origin->y);
    }
    float dx = (dst->x - origin->x);
    float dy = (dst->y - origin->y);
    return sqrtf(dx * dx + dy * dy);
}
```

This also matches the original code.

### Delay Slots and `NOP`s

#### `NOP`s due to floating point literals (PS2)

This behavior was found in GCC 2.9 991111 (PS2) in Twisted Metal: Black, and
likely only applies to PS2 due to the use of the `.lit4` section.

By default on PS2, GCC will remove most floating-point literals from the code
and relocate them into the `.lit4` section, where they can be loaded from
memory.

When matching the code before `.lit4` migration, the literals are typically
referenced as `extern float` variables until the TU has been fully decompiled,
at which point they can be replaced with the original literals.
Take `mathfRPHFromMatrix()`:

```c++
extern float D_004FA668; // 1.5707964f
extern float D_004FA66C; // -1.5707964f

void mathfRPHFromMatrix(FMATRIX mat, FVECTOR* result)
{
    // ...
    
    if (mat[1][2] > 0.0f) {
        result->x = D_004FA668; // pi / 2
        result->z = atan2f(-mat[0][1], -mat[2][1]);
    } else {
        result->x = D_004FA66C; // -pi / 2
        result->z = atan2f(-mat[0][1], mat[2][1]);
    }
    result->y = 0.0f;
}
```

However, the `.lit4` substitution can change codegen.
When compiled, the code is actually missing a `nop` compared to the original:

```
// This is the `if (mat[1][2] > 0.0f) {}` branch check.
TARGET                                                   CURRENT (60)
dbcf4:    mtc1    zero,$f0                               dbcf4:    mtc1    zero,$f0                 
dbcf8:    nop                                            dbcf8:    nop                              
dbcfc:    c.lt.s  $f0,$f1                                dbcfc:    c.lt.s  $f0,$f1                  
dbd00:    nop                                            dbd00:    nop                              
dbd04:    bc1f    dbd24 ~>                               dbd04:    bc1f    dbd20 ~>                 
dbd08:    nop                                     <                                                 
dbd0c:    lwc1    $f0,%gp_rel(D_004FA668)(gp)            dbd08:    lwc1    $f0,%gp_rel(D_004FA668)(gp)          
dbd10:    swc1    $f0,0(s1)                              dbd0c:    swc1    $f0,0(s1)      
```

The `nop` is included as expected if we properly substitute the literals.

```c++
void mathfRPHFromMatrix(FMATRIX mat, FVECTOR* result)
{
    // ...
    
    if (mat[1][2] > 0.0f) {
        result->x = 1.5707964f; // pi / 2
        result->z = atan2f(-mat[0][1], -mat[2][1]);
    } else {
        result->x = -1.5707964f; // -pi / 2
        result->z = atan2f(-mat[0][1], mat[2][1]);
    }
    result->y = 0.0f;
}
```

Unfortunately, the only workaround for this when matching a TU is to leave the
function alone until you're ready to migrate and match the `.lit4` section;
you won't be able to compile otherwise.

### C++ Features

#### `bool` load/store order

This behavior was found in GCC 2.9 991111 (PS2) in Twisted Metal: Black.

In old GCC compilers with C++ support, the C++ `bool` type can behave differently
from an `int` used in the same location, particularly with respect to variable
loads and stores. Take `Combo::Update()`:

```c++
class Combo {
public:
    int state;
    int pad_index;
		// ...
    void Update(Vehicle* vehicle);
    // ...
};

void Combo::Update(Vehicle* vehicle)
{
		// ..
		this->rightPressed_buf[frame] = rightPressed;
    // Handle analog stick inputs.
    joyStick = inputFixAnalogValue(2, this->pad_index);
    // ...
}
```

This produces a mismatch on the call to `inputFixAnalogValue`; the variable
load for `this->pad_index` is swapped.

```
1c8d4:    sw      t0,0x214(a1)                           1c8d4:    sw      t0,0x214(a1)             
1c8d8:    lw      a1,4(s0)                        |      1c8d8:    sw      v0,0x304(v1)             
1c8dc:    jal     1d8da8                                 1c8dc:    jal     1d8da8                   
1c8e0:    sw      v0,0x304(v1)                    |      1c8e0:    lw      a1,4(s0)                 
1c8e4:    move    v1,v0                                  1c8e4:    move    v1,v0         
```

However, if we change the type of `pad_index` from `int` to `bool`, the code
matches without issue.

```c++
class Combo {
public:
    int state;
    bool pad_index;
		// ...
    void Update(Vehicle* vehicle);
    // ...
};
```

```
1c8d4:    sw      t0,0x214(a1)                           1c8dc:    sw      t0,0x214(a1)             
1c8d8:    lw      a1,4(s0)                               1c8e0:    lw      a1,4(s0)                 
1c8dc:    jal     1d8da8                                 1c8e4:    jal     1d8da8                   
1c8e0:    sw      v0,0x304(v1)                           1c8e8:    sw      v0,0x304(v1)             
1c8e4:    move    v1,v0                                  1c8ec:    move    v1,v0          
```

