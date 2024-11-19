---
title: Decomp Intro (Melee)
description: 
published: true
date: 2024-11-19T18:33:28.731Z
tags: 
editor: markdown
dateCreated: 2024-11-19T18:33:28.731Z
---

# Intro to Matching Decompilation

The goal of this document is to present a holistic overview of the decomp scene and how the process of decompilation works, hopefully allowing someone who has close to zero knowledge of decomp to quickly get up to speed and gain the confidence to start contributing to projects. I'll also be walking you through how to use the web-based decomp.me to decompile your first function, with no external programs or downloads required. This guide currently focuses on Melee for now, but most of the information in this section is useful to all GameCube and Wii projects. And, as always, reach out on the [Discord server](https://discord.gg/hKx3FJJgrV) if you have any questions or comments!

## Prerequisites

Decompilation is somewhat unique in that it's more like a puzzle than a concrete "programming" problem; you don't necessarily have to have any experience in graphics programming or algorithms or game engine development to be effective at doing it. The only knowledge that would be *baseline* for you to understand most of this article would be surface-level knowledge of the C programming language (i.e. pointers, functions, structs) and how the CPU works (i.e. what assembly is, what are registers, the flag bits on the CPU). A fun way to learn about the latter is to follow a tutorial on making a Game Boy emulator, specifically implementing its 8080-like CPU. There's an [emulator developers Discord server](https://discord.gg/emudev) where you can get support on how to write one if you're interested. As long as you have some semblance of knowledge of these things, you should be alright.

## What is "decomp?"

You, having found this document, likely already have a general idea of what decomp is and the games that have currently been decompiled fully. For the most part, the goal of a decomp project in the GC/Wii server is to create a *matching decomp*, a functionally identical reconstruction of the game code from the machine code that the game shipped on. Ideally we want to write source code that compiles into a byte-for-byte copy of the original game, so that we know for certain that our implementation of the code is correct. However in practice each game will have its own unique circumstances and challenges that change the specifics of how this "matching" is done. Some examples:
- Melee has a fair bit of unused code that still got linked into the final game, namely a bytecode interpreter that's several KB large, that we won't count as part of "completing" the project. 
- The repo members on Mario Kart Wii aren't requiring regswaps to be matched (regswaps will be discussed later), opting to use an equivalency checker to prove that they still match the functionally of the source assembly 1:1.
- Modern compilers are a lot more lenient in how code is compiled, so projects like Breath of the Wild have to worry a lot less about how the original developers may have written the code. They still write code that matches the assembly of each function exactly, but the functions won't be put in exactly the same spots in memory as the original game. [This page](https://botw.link/about) has a great writeup on that.
- Developers may have compiled the final game or an early demo in an unoptimized debug state, which makes the assembly a lot easier to match. This is notably true for Mario 64 and Mario Party 4, the GC decomp with the most progress. 
- A game may additionally have partial or full debug *symbols* (discussed later), which reveal things to us like translation unit splits (more on that later), function arguments, and struct members without having to guess. 
- Since many games during a given time period make use of the same libraries, progress/symbols from one game are often carried over to another. 

## The PowerPC Architecture

PowerPC (PPC) is an instruction set architecture (ISA) that used to be used in computers and game consoles in the 90s and 2000s, notably in several Apple products, the Xbox 360, the PS3, and the GC/Wii/Wii U. Nowadays x86 and ARM are the predominant ISAs in terms of consumer hardware, though you don't need specialized training on PowerPC specifically it to be able to do decomp. Specifically, the GameCube and Wii use an embedded, 32-bit version of PowerPC on their CPUs. There are 32 registers for general purpose operations (GPRs, r0..r31) and 32 registers for floating point operations (FPRs, f0..f31). The size of the GPRs are 4 bytes on 32-bit and 8 bytes on 64-bit, while FPRs are 8 bytes on both types. This means that 64-bit integers generate assembly that's a bit different from what you'd expect, which will be explored later.
The resources relevant to Melee include:
- [PowerPC User Instruction Set Architecture](https://files.decomp.dev/ppc_isa.pdf), which is useful for looking up what an opcode does. 
- [System V Application Binary Interface PowerPC Processor Supplement](http://refspecs.linux-foundation.org/elf/elfspec_ppc.pdf) and the [PowerPC Embedded Application Binary Interface (EABI): 32-Bit Implementation](https://www.nxp.com/docs/en/application-note/PPCEABI.pdf), which is useful for understanding what each register does. 
- [The PowerPC Compiler Writer’s Guide](https://files.decomp.dev/IBM_PPC_Compiler_Writer's_Guide-cwg.pdf), which is useful to understand specific compiler behavior like integer division.

## Decomp and the compiler

This being a matching decomp means that we want to have as close to the exact same compiler version and project settings that the developers used as possible. Determining this information from the machine code as well as locating the compiler itself can become a whole ordeal in and of itself, where in the case of games before the late 2000s, also entails buying as many old developer CDs as possible and hoping it contains the exact compiler version that's desired. 

The PowerPC compiler that Melee used was made by a company called Metrowerks, one of a few compilers used by GameCube devs at the time. This is commonly referred to as MWCC (MetroWerks C Compiler), which is bundled with an IDE also made by Metrowerks called CodeWarrior (CW). In the case of Melee, this version is believed to be a special hotfix of version 1.2.5 that fixed a minor compiler bug and was available on a Metrowerks FTP server at some point. While we don't have this specific revision of the compiler, a decomp member named ninji was able to successfully patch the base 1.2.5 to produce a compiler that should be able to be used to match the game fully, known unofficially as `1.2.5n`.

## Important topics

One of the most important things to be aware of to make sense of how decompiling works is to understand how functions in C/C++ get compiled and how building a C/C++ program in general happens. 

### Functions

For starters, consider these two functions and the PowerPC disassembly it would compile to if you were to inspect a game's executable directly in Dolphin:


```c
int adder(int a, int b)
{
    return a + b;
}

s32 call_adder()
{
    return adder(7, 9);
}
```

```
802d8618    add r3, r3, r4
802d861c    blr
802d8620    mflr    r0
802d8624    li  r3, 0x7
802d8628    stw r0, 0x0004 (sp)
802d862c    li  r4, 0x9
802d8630    stwu    sp, -0x0008 (sp)
802d8634    bl  ->0x802d8618
802d8638    lwz r0, 0x000C (sp)
802d863c    addi    sp, sp 8
802d8640    mtlr    r0
802d8644    blr
```

Note that I've purposely compiled this to prevent auto inlining and make both functions appear discretely, with `adder` corresponding to the first two lines and `call_adder` being the rest. You'll see how auto inlining can "remove" where a function would be later. First I'll step through and explain each instruction individually, starting with where `call_adder()` starts, at the `mflr`:
- `802d8620    mflr    r0`
    - This stores the contents from the link register into r0. You'll see why the function has to do this in a second.
- `802d8624    li  r3, 0x7`
    - This loads 7 into r3, which corresponds to the first argument in `adder(7, 9)` and will be used by `adder` later. 
- `802d8628    stw r0, 0x0004 (sp)`
    - This relates to setting up the stack that's not relevant to discuss now.
- `802d862c    li  r4, 0x9`
    - This serves the same function as the previous `li`, loading the 9 into r4 to be used.
- `802d8630    stwu    sp, -0x0008 (sp)`
    - More irrelevant stack setup. Note that the `li` instructions and the next `bl` instruction that call `adder` with the 7 and 9 don't necessarily have to appear contiguously, and the compiler is allowed to interleave other operations between them as long as they're not dependent on each other.
- `802d8634    bl  ->0x802d8618`
    - This makes the CPU *branch* to specified address and start executing the opcodes from there, which in this case (`0x802d8618`), is the beginning of our `adder` function. It also overwrites the link register with the next instruction after itself, `802d8638    lwz r0, 0x000C (sp)`.
- `802d8618    add r3, r3, r4`
    - As you may be able to guess, this adds r3 and r4 together and stores the result in r3. From this instruction, we can deduce the following, which you can also verify on page 3-14 of the System V ABI for PowerPC:
        - In order to pass non-floats to a function, you must fill out r3 through r10. For floats, it's f0-f8.
        - If a function returns a non-float value, it will use r3 to do so (r4 can also be used in conjunction with r3 to return a 64-bit value). For floats, it's f1 only (float registers are already 64 bits).
- `802d861c    blr`
    - This makes the CPU branch to the address in the link register, which `blr` previously filled with the `lwz` op.
- `802d8638    lwz r0, 0x000C (sp)`
- `802d863c    addi    sp, sp 8`
    - The cleanup routine for the stack.
- `802d8640    mtlr    r0`
    - This loads the contents from r0 into the link register. We stored the contents of the link register at the beginning of the function, so the link register now points to whatever function elsewhere in the code called `call_adder`.
- `802d8644    blr`
    - This breaks to whatever function called `call_adder`, and ends execution of `call_adder`. From these last two instructions we can deduce the following:
        - A function will use `mflr` and `mtlr` if it has to call a function and modify the link register, which is why `adder` doesn't need to do so.
        - We can tell where functions start and stop without looking at the source code by looking between pairs of `blr` opcodes.

You likely won't fully grasp these concepts if you're still new to assembly and haven't done much decomp yet, but the main takeaway here is that functions in C are simply an address in memory that a caller breaks to. 

### Compiling and linking

When we say that a C program gets "compiled" or invoke a compiler like MWCC or MSVC or Clang or GCC, there’s actually two processes happening: a compiler *and* a linker. It may be a bit fuzzy to you even if you have C experience if you've only ever invoked both stages at the same time or only use tools like CMake or Ninja, where the actual compilation command that gets executed is hidden from you by default. For example, let's say we have a typical C/C++ project that consists of two files named `file_a.c` and `file_b.c`, and we want to compile it into an executable with gcc:

```
gcc -o out file_a.c file_b.c
```

This will do what you expect, outputting an executable named `out`. If we wanted to invoke the compiling and linking stages separately, we would do this:

```
gcc -c file_a.c
gcc -c file_b.c
gcc -o out file_a.o file_b.o
```

The first two commands use the `-c` flag to invoke just the compiler and output what are referred to as *object files* with a default name of `file_a.o`, `file_b.o`. We then pass both of these to gcc with no special arguments and it will implicitly see that there are no source files to compile and only run the linker. From this it's important to note these observations:
- Each source C/C++ file that gets passed in as an argument to the compiler is called a translation unit. Only the file itself is counted as a TU, and any i.e. header files `#include`d in the C file are a part of that same TU.
- Note that it's only the *linker* that needs to be able to see "everything" to produce an executable, not the *compiler*. The compiler can work on multiple TUs in parallel, because compilation and the resulting object file only concerns the content of each individual TU. 
- The machine code in object files are for the most part the same as what appears in the finalized executable, with the key differences arising from the limitation that each object file exists independently from each other.

To make the last point more clear, here's an example with the same functions `adder` and `call_adder` from above, but we're looking at the disassembly of the machine code from the resulting *object files*:

```c
int adder(int a, int b)
{
    return a + b;
}

s32 call_adder()
{
    return adder(7, 9);
}
```

```
00    add   r3, r3, r4
04    blr
08    mflr    r0
0C    li  r3, 0x7
10    stw r0, 0x0004 (sp)
14    li  r4, 0x9
18    stwu    sp, -0x0008 (sp)
1C    bl    adder
20    lwz   r0, 0x000C (sp)
24    addi    sp, sp 8
28    mtlr    r0
2C    blr
```
(There is more data in the object file than just machine code, for now that's all that's being shown)

It's identical to the code in the executable, with only two differences:
1. The addresses start at a relative offset to the object file (`adder` and `call_adder` being the first two functions defined) rather than offset to an actual memory address. This is because the compiler can't predict the size of every other TU ahead of time, and it's the linker's job to glue all of the TUs together and change these to finalized addresses.
2. The `bl` instruction has been replaced with what's referred to as a *symbol* to a function called `adder`. This is again because of the above condition. In this case since the functions share the same TU you could argue something like "offset 0" instead of the name specifically would work, but remember the compiler typically would not emit this had I not disabled inlining here. If I were to compile this with inlining, the functions would look like this:

```
00    add   r3, r3, r4
04    blr
08    li    r3, 0x10
0C    blr
```

`adder` doesn't change, but the nine instructions involved in setting up the stack and passing arguments to `adder` in `call_adder` have now become a single load into the return register! (0x10 being 16, the result of 7+9 already calculated). You're probably wondering why the code for `adder` is even still generating now that the `bl    adder` has been optimized away, and if you've been paying very acute attention, you probably already know the answer. It's because since only the *linker* gets to see all the object files, the *compiler* can't assume `adder` isn't called anywhere else! If you were to run this object file through the linker and inspect the address where `adder` would be in the compiled executable, however, you'd see that it's gone:

```
802d8618    li    r3, 0x10
802d861c    blr
```
and that's because the linker actually does gain the ability to delete a function in the linking stage if it sees that no object file actually calls it. Some other relevant tidbits:
- If `adder` was in a different TU from `call_adder`, then `call_adder` would generate exactly how it does in the first version with auto inlining disabled, because the compiler now can't see what `adder` is when compiling `call_adder`.
- If I had used the `static` keyword on `adder`, which (when used on a function) means you're telling the compiler that no other TU uses that function, then the compiler would have actually been able to delete the machine code in the object file for that function *before* the linking stage. This is why you get a *linker* error when calling a function from another TU that has the `static` keyword instead of a *compiler* error. The compilation stage can't know there's a problem until it gets to linking!
- The `inline` keyword has gained less and less relevance to compilers as they get "smarter" with their optimizations, with compilers even in the 2000s like MWCC often making the choice whether to inline a function or not regardless of the keyword's presence. It still has some influence in MWCC and the codegen in Melee, however, and shouldn't be overlooked when working on a file.

## Aside on types of executables

The way Melee as a program is structured is similar to how I've been talking about the compiler and linker so far: you gather up all your object files and turn them into one big executable file that runs the game. Now that's not the *only* model of compilation a game can opt to use. A big disadvantage with having all of your code in one executable is that the GC/Wii has to have that entire codebase in RAM at all times. Remember that the GC only has 24 MB of RAM (for a few smart developers it's a bit more, but most chose to only use 24) and the Wii which has 88 MB. Melee's executable file is 3 MB, which is miniscule by today's standards, but that's about 13% of the 24 MB of RAM. On a modern machine with 16 GB of RAM, that would be like if a program just ate 2 GB of your RAM for no reason! (Which isn't that far from the truth if you open a couple of Chrome tabs...) So it's kind of impressive that the Melee devs just didn't feel the need to use that extra 10% of RAM. 

Instead you can use what are called *relocatable* object files, which in the case of the GC/Wii, are mostly REL ("relocatable") files. Basically you can choose to just not run the linker on certain object files when making the executable and instead, when the game is being run, have the OS perform a kind of "linking" on demand that allows you to only put the code you actually need into RAM. You may be familiar with DLL, the Windows equivalant of a REL, and if you think about what it stands for, "dynamically linked library," it's pretty much just that. 

Of course executable size isn't the only reason a developer might opt to use a REL, but for some games it's a big deal. Brawl, for instance, breaks up every stage and fighter into its own REL that get loaded/unloaded on demand. There's pros and cons to this choice of program layout when it comes to decompilation, but a great thing about it in terms of modding is that we can add/remove content to a REL without needing to offset the rest of the program. If you recall how `adder()` in the compiled executable started at 802d8618 versus at the top of its object file with a local address of 0, you can probably piece together why this is. If I wanted to put a new function before `adder()` that's 20 bytes, I'd have to shift `adder()` and every function below it down 20 bytes, which would also require changing opcodes like `bl  ->0x802d8618`. On the other hand if `adder()` was in a relocatable object file, well the address of `adder()` isn't decided in memory, so every function that calls `adder()` in the base executable instead jumps to a spot that the OS will fill with the code for `adder()`, so we can modify the DLL all we want without affecting the rest of the game. It's one reason among others that Brawl modding became much more prolific than Melee.


## The decomp process

Now we can finally bring it around and talk about how the decompilation process works. To explain it visually, basically, we have this:

```mermaid
flowchart LR
    A["???"]-->A1["???"]
    subgraph "Compiler (MWCC)"
    A1
    end
    A1-->A2["???"]
    A2-->L["???"]
    subgraph "Linker (MWCC)"
    L
    end
    L-->F[main.dol]
    subgraph Melee disc
    F
    end
```

and we want to end up with this:

```mermaid
flowchart LR
    A[file_a.c]-->A1[Compile file_a.c]
    B[file_b.c]-->B1[Compile file_b.c]
    C[file_c.c]-->C1[Compile file_c.c]
    D[file_d.c]-->D1[Compile file_d.c]
    subgraph "Compiler (MWCC)"
    A1
    B1
    C1
    D1
    end
    A1-->A2[file_a.o]
    B1-->B2[file_b.o]
    C1-->C2[file_c.o]
    D1-->D2[file_d.o]
    A2-->L[Link into executable]
    B2-->L
    C2-->L
    D2-->L
    subgraph "Linker (MWCC)"
    L
    end
    L-->F[main.dol]
    subgraph Melee disc
    F
    end
```

and in order to do that, we do this:

```mermaid
flowchart LR
    F[main.dol] --> L[Split into object files]
    subgraph "&quot;Un&quot;-linker (dtk)"
    L
    end
    L --> A2[file_a.o]
    L --> B2[file_b.o]
    L --> C2[file_c.o]
    L --> D2[file_d.o]
    subgraph "&quot;De&quot;-compiler (m2c, you)"
    A1
    B1
    C1
    D1
    end
    A2-->A1[Decomp file_a.o]
    B2-->B1[Decomp file_b.o]
    C2-->C1[Decomp file_c.o]
    D2-->D1[Decomp file_d.o]
    A1 --> A[file_a.c]
    B1 --> B[file_b.c]
    C1 --> C[file_c.c]
    D1 --> D[file_d.c]
```

So we have the executable and the compiler/linker, and we want to write C code and run it through the original compiler that the HAL Laboratory devs used to produce more or less the same object files that get linked by the original linker into a bit-for-bit recreation of the executable.

Our first step would be to "undo" the linking step and produce ad hoc object files that are *similar* to what the Melee devs had, but as you saw earlier with function inlining, we can't losslessly restore what the original objects might have looked like from the executable alone. Primarily this process involves identifying *splits* where one object file begins and another ends, creating *symbols* for every function and global variable, and changing opcodes that reference constant addresses like the `bl  ->0x802d8618` back into a label. This is a bit of work depending on the compiler (especially if you don't have debug symbols or a similar game's code to work off of), but with PowerPC and Metrowerks specifically, it isn't that hard. It's fairly simple to programmatically analyze an executable of this type and guess where splits, functions, and global variables are with a high degree of confidence, which is exactly what the program [dtk](https://github.com/encounter/decomp-toolkit) does. Some other asides on this topic:
- Whenever you write `x = 0.0f` or `y = 1.0f` in a TU, that floating point value has to get saved in the object file somewhere. A very fortunate aspect of the Metrowerks linker is that if it encounters multiple object files that have, say, 0.0f or 1.0f defined in them, it doesn't try to remove, or *deduplicate* these extra values and instead just packs them together as-is, allowing us to identify where a TU might start by looking for each 0.0f or 1.0f.
- I mentioned in the functions section how `adder` can only get inlined into `call_adder()` if both were part of the same TU. This is because typically it's not the linker's role to perform optimizations, but if you were to turn on what is called *link-time optimization* on a modern compiler, heuristics like inlining only being an inner-TU operation and object files being similar to the final executable become false. It suddenly becomes massively harder to approach a decomp with this enabled, and this is why decompiling the latest version 1.6.0 of Breath of the Wild, for instance, is considered [a lost cause](https://botw.link/about#why-not-decompile-160).

The next step, which is where we are today, is to produce C code that compiles into a matching copy of these object files. It is arguably the most difficult part of completing a decomp project due to the sheer amount of code that goes into a game and the time it takes to make sure functions match exactly. It's one of the reasons this guide and sites like decomp.me exist: to make it easy for many people to collaboratively take on this problem. The rest of this article will concern this process and show you how to contribute to the decomp, and now that you are hopefully better-informed as to the inner workings of the decomp process, you'll be able to approach this with a lot better intuition.

## How to decomp

Your biggest ally when it comes to completing a game's decomp besides the general community is going to be the tooling built by talented individuals in that community around that game/platform. You've already learned about one such example with dtk, which deals with the splitting and symbols stage, and you've probably at least seen decomp.me, which deals with compiling C code and comparing the resulting object file with the one to match. I'll briefly mention some other relevant tools before going into detail on [decomp.me](https://decomp.me):
- [m2c](https://github.com/matt-kempster/m2c)
    - The C decompiler that decomp.me runs behind the scenes when you first create a scratch or hit the "Decompile" button. It supports compilers for both the MIPS architecture (used by the N64) and the PowerPC architecture.
- [objdiff](https://github.com/encounter/objdiff)
    - A UI and CLI tool that allows the quick comparison of symbols, instructions, and other data sections between two object files, basically a more complete version of the righthand UI from decomp.me.
- [dtk-template](https://github.com/encounter/dtk-template)
    - A convenient project workflow for starting a Wii/GC decomp that primarily uses dtk, objdiff, and ninja. The Melee repo is one such project that uses this.
- [Ghidra](https://github.com/NationalSecurityAgency/ghidra) and [IDA](https://hex-rays.com/)
    - Two IDE-like tools for reverse engineering, used in a fair bit of decomps.

(to be continued)
