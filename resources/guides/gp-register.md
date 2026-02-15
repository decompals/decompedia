---
title: The $gp register
description: gp-register
published: true
date: 2026-02-15T19:39:35.231Z
tags: 
editor: markdown
dateCreated: 2025-01-29T15:17:48.716Z
---


# The `$gp` Register in MIPS  

The MIPS architecture (particularly relevant for N64, PS1, and PS2 consoles) often requires two instructions to load data from memory, e.g., `lui` + `lw`.  

This is because memory addresses are 32-bit, while immediate operands in MIPS instructions are only 16-bit.  

## Standard Memory Access in MIPS  
To load a word from `0x80201234`, two instructions are required:  

```assembly
lui  $v0, 0x8020      # Load upper 16 bits of the address
lw   $v0, 0x1234($v0) # Load from the full address
```  

Compilers can optimize by reusing the high address if only the low bits change, but in general, most memory loads require two instructions.  

## How `$gp` Optimizes Memory Access  
The `$gp` (Global Pointer) register allows for single-instruction memory loads by providing a base address for accessing small global/static data:  

```assembly
lw   $v1, 0x1234($gp)  # Load from an offset within the small data section
```  

This works because `$gp` is conventionally initialized to a fixed memory location, typically near the middle of the **small data section**, allowing efficient access to variables within ±32 KB (commonly ±0x8000 bytes).  

## Determining initial value of `$gp`

The value for `$gp` is set in the entrypoint for the game, it will look something like this:
```assembly
/* A7738 800B7138 0E801C3C */  lui        $gp, (0x800E0000 >> 16)
/* A773C 800B713C 90409C27 */  addiu      $gp, $gp, 0x4090
```
In this example the `$gp` register is initialised to `0x800E4090`.

## Compiler Usage of `$gp`  
To instruct the compiler to use `$gp`, a flag is used:  
- **GCC:** `-G<size>`  
- **MWCC:** `-sdatathreshold <size>`  

The threshold determines the maximum size (in bytes) of variables that can be placed in the small data section. The default is typically **8 bytes**, meaning that variables **≤8 bytes** (e.g., `char`, `short`, `int`, `int64_t`) are stored in:  
- `.sdata` (for initialized data)  
- `.sbss` (for uninitialized data)  

The **`s` prefix** stands for **"small"**, distinguishing these sections from `.data` and `.bss`.  

## Compiler Differences in `$gp` Usage  

- **GCC:** Uses `$gp` **only** for variables defined within the current **translation unit (TU)**. If a variable is declared `extern`, GCC falls back to the traditional **two-instruction** access (`lui + lw`).  
- **MWCC:** Smarter about `$gp` usage and will attempt to use `$gp` even for `extern` variables, provided the linker places them within range.  

## Relevance for Reverse Engineering  

In decompilation, GCC's `$gp` behavior provides useful hints:  
- If a variable is accessed via **both** `$gp` and a traditional `lui/lw` sequence within the same file, it's likely that the code was originally split across multiple source files.  
- This can be used as an indicator when reconstructing original file boundaries in projects.  