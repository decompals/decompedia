---
title: PS1
description: PlayStation 1
published: true
date: 2025-10-21T16:42:03.782Z
tags: 
editor: markdown
dateCreated: 2024-11-25T20:54:39.020Z
---

# PSX

## Compilers

The vast majority of PlayStation games use GCC-derived compilers as this was the compiler shipped with the official toolchain (PSYQ/SDevTC), however it's not impossible for games to have used a different compiler as is the case with Tokimeki Memorial (RISCompiler/IRIX-derived compiler) or Digimon World (Codewarrior).

The existing tools designed to support N64 decompilation, such as **m2c**, **asm-differ**, and **decomp-permuter**, are built to work with ELF object files. Since the PSYQ toolchain generates proprietary object files, tools have been created to convert these into standard ELF objects. Alternatively, projects like **maspsx** can be used to produce output identical to PSYQ compilers and assemblers, but natively in ELF format.

- PSYQ/SDevTC (GCC)
- CodeWarrior (MWCC)
- RISCompiler

## Tools

- [maspsx](https://github.com/mkst/maspsx)
- [psyq2elf](https://gitlab.com/jype/psyq2elf)
- [psyq-obj-parser](https://github.com/grumpycoders/pcsx-redux/tree/main/tools/psyq-obj-parser)

### Decompilers

#### Ghidra

A fully-fledged executable analysis tool. It is open source.

You may want to use the [ghidra_psx_ldr](https://github.com/lab313ru/ghidra_psx_ldr) plugin, which automatically creates the right memory regions and comes with a way to regonize the right PSY-Q SDK by matching the libraries with their signatures.

The decompiler produces noisy but somewhat reliable output.

#### IDA Pro

Paid product. A MIPS decompiler was introduced starting from **IDA Pro 7.5**. The output looks clean but often over-simplified. Certain instructions are completely absent from the output; this is due to the disassembler outputting pseudo-instructions that combine multiple instructions (such as outputting a pseudo instruction for `lw` from a `lui` and `lw`). You may want to use the [psxida]( https://github.com/lab313ru/psxida/) plugin. Not recommended other than very specific scenario.

**NOTE::** You can get around the truncation by going to "Options->General->Analysis->Processor specific analysis options" and unchecking "Simplify instructions".