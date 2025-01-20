---
title: DLLSIMPORT.tab
description: 
published: true
date: 2025-01-20T01:36:48.831Z
tags: 
editor: markdown
dateCreated: 2025-01-20T01:36:48.831Z
---

# DLLSIMPORT.tab
The DLLSIMPORT.tab file contains a mapping of indexes to symbol addresses pointing to core code VRAM. When a DLL references an external symbol, the symbol address is resolved through this file. External symbol references are listed in a DLL's global offset table with a value that is a 1-based index into DLLSIMPORT.tab additionally with its 32nd bit set (0x80000000). For example, a GOT entry of `0x80000004` references the 4th DLLSIMPORT.tab entry.

At runtime when a DLL is relocated, its external "import" entries are replaced with the corresponding address found in DLLSIMPORT.tab.

## Entries
DLLSIMPORT.tab has no header and instead is simply an unterminated list of VRAM addresses:

| Offset | Name | Type | Description |
|--------|------|------|-------------|
| 0x0 | address[0] | u32 | The core code VRAM address for the current index. |
| ... | ... | ... | ... |
