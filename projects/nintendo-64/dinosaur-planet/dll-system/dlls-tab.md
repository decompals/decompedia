---
title: DLLS.tab
description: 
published: true
date: 2025-01-20T01:41:32.997Z
tags: 
editor: markdown
dateCreated: 2025-01-20T01:36:08.999Z
---

# DLLS.tab
The DLLS.tab file contains a table listing the regions of [DLLS.bin](/projects/nintendo-64/dinosaur-planet/dll-system/dll-format) that contain each DLL as well as the `.bss` size of each DLL.

## Header
The file starts with a header as follows:

| Offset | Name | Type | Description |
|--------|------|------|-------------|
| 0x0 | bank1Index | s32 | Entry index where bank 1 starts. |
| 0x4 | bank2Index | s32 | Entry index where bank 2 starts. |
| 0x8 | - | s32 | Unknown. |
| 0xC | bank3Index | s32 | Entry index where bank 3 starts. |

## Entries
After the header is a list of DLL entries terminated by `0xFFFFFFFF_FFFFFFFF` at the end of the file:

| Offset | Name | Type | Description |
|--------|------|------|-------------|
| 0x10 | entry[0] | DLLSTabEntry | A DLL entry. |
| ... | ... | ... | ... |
| - | - | - | 0xFFFFFFFF_FFFFFFFF |

Where each entry (`DLLSTabEntry`) is:

| Offset | Name | Type | Description |
|--------|------|------|-------------|
| 0x0 | offset | s32 | Offset of DLL in DLLS.bin. |
| 0x4 | bssSize | s32 | Number of bytes to be allocated for the DLL's `.bss` segment.

> The final entry is not an actual DLL. Instead, it exists just to provide the end offset for the previous entry.
{.is-info}

## Banks
DLLs are split up into four banks. The ID of a DLL contains both its bank and its index within the bank.

IDs are mapped to each bank as follows:

| ID Range | Bank |
|----------|------|
| [0, 0x1000) | 0 |
| [0x1000, 0x2000) | 1 |
| [0x2000, 0x8000) | 2 |
| [0x8000, EOF) | 3 |

For example, an ID of 0x2004 would be a DLL in bank 2 with an index of 4 (`0x2004 & (~0x2000)`).
