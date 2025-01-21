---
title: DLL Format
description: 
published: true
date: 2025-01-21T08:37:46.960Z
tags: 
editor: markdown
dateCreated: 2025-01-20T01:32:52.042Z
---

# DLL Format
Each DLL is found inside of DLLS.bin. The file itself does not contain a header or a way to locate a DLL inside. Instead, the individual DLLs can be located by using the information in [DLLS.tab](/projects/nintendo-64/dinosaur-planet/dll-system/dlls-tab) which contains offsets for this file.

An individual DLL contains the following:

- A header
- An export table
- A `.text` section
- A `.rodata` section (optional)
- A `.data` section (optional)

## Header
A DLL file starts with a header. The header lists offsets to each section, the number of exports, and offsets to the DLL's constructor and destructor functions.

| Offset | Name | Type | Description |
|--------|------|------|-------------|
| 0x0 | textOffset | u32 | Byte offet to `.text` section. |
| 0x4 | dataOffset | u32 | Byte offet to `.data` section (or `0xFFFFFFFF` if it doesn't exist). |
| 0x8 | rodataOffset | u32 | Byte offet to `.rodata` section (or `0xFFFFFFFF` if it doesn't exist). |
| 0xC | exportCount | u16 | Number of functions this DLL exports. |
| 0x10 | ctorOffset | u32 | Byte offset from `.text` to the constructor function. |
| 0x14 | dtorOffset | u32 | Byte offset from `.text` to the destructor function. |

## Exports
Following the header is the DLL's export table. This defines the "public interface" of the DLL, allowing outside code to invoke functions inside of the DLL through the export table.

Before the actual export entries is a single 32-bit number with a currently unknown purpose. It's typically zero. After that is a list of export table entries, with a length of `exportCount` from the header.

| Offset | Name | Type | Description |
|--------|------|------|-------------|
| 0x18 | unknown | u32 | Unknown. |
| 0x1C | exportOffset[0] | u32 | Byte offset from `.text` to the exported function. |
| ... | ... | ... | ... |

## .text Section
The DLL's `.text` section contains all of the DLL's executable code. This is position-independent code (PIC) making use of the `$gp` register. After a DLL is relocated, each function (that needs it) starts by setting the `$gp` register to the current address of the DLL's global offset table . This allows the function to reference other functions and data outside of the function in a position-independent way.

## .rodata Section
The `.rodata` section is special as it contains not only the DLL's readonly data but also its global offset table and relocation table.

| .rodata Visualization |
|-|
| Global offset table |
| 0xFFFF_FFFE |
| $gp relocations |
| 0xFFFF_FFFD |
| .data relocations |
| 0xFFFF_FFFF |
| Readonly data... |

### Global Offset Table
The start of `.rodata` is always the global offset table. Before relocation, this is a list of byte offsets from `.text` for local symbols and DLLSIMPORT.tab indexes for external symbols. After relocation, this is a list of absolute VRAM addresses to each symbol. The GOT is terminated by `0xFFFF_FFFE`.

The first four GOT entries are always for `.text`, `.rodata`, `.data`, and `.bss` in that order. If a section is not present, its GOT entry will simply point to the start of the next section as if it had a size of zero.

| Offset | Name | Type | Description |
|--------|------|------|-------------|
| 0x0 | textOffset | u32 | Byte offset from `.text` to `.text`... always zero. |
| 0x4 | rodataOffset | u32 | Byte offset from `.text` to `.rodata`. |
| 0x8 | dataOffset | u32 | Byte offset from `.text` to `.data`. |
| 0xC | bssOffset | u32 | Byte offset from `.text` to `.bss`. |
| 0x10 | gotEntry[4] | u32 | If the 32nd bit is set (0x80000000), a [DLLSIMPORT.tab](/projects/nintendo-64/dinosaur-planet/dll-system/dlls-import-tab) index, otherwise a byte offset from `.text` to the symbol.
| ... | ... | ... | ... |
| - | - | - | 0xFFFF_FFFE |

### Relocation Table
Immediately following the global offset table is the DLL's relocation table. This is split into two parts: `$gp` prologue relocations and `.data` relocations.

First is a list of byte offsets from `.text` to the `$gp` prologue of each DLL function that accesses the GOT. At relocation time, these prologues will be updated so that they initialize `$gp` to the current absolute address of the DLL's GOT. This list is terminated by `0xFFFF_FFFD`.

| Offset | Name | Type | Description |
|--------|------|------|-------------|
| 0x0 | gpPrologueOffset[0] | u32 | Byte offset from `.text` to a function's `$gp` prologue. |
| ... | ... | ... | ... |
| - | - | - | 0xFFFF_FFFD |

Immediately following the `$gp` prologue relocations are the `.data` relocations. These relocations point to values in the `.data` section that need to be offset by the relocated address of the DLL. An example would be a global variable initialized to the address of one of the DLL's functions.

| Offset | Name | Type | Description |
|--------|------|------|-------------|
| 0x0 | dataOffset[0] | u32 | Byte offset from `.data` to a data value that needs to be relocated. |
| ... | ... | ... | ... |
| - | - | - | 0xFFFF_FFFF |

## Non-standard $gp Prologue
Dinosaur Planet DLLs do not use standard MIPS `$gp` prologues:
```mips
; Dinosaur Planet's version
; Note that it uses the ori instruction instead of addiu
lui $gp, 0x0
ori $gp, $gp, 0x0
nop
```
