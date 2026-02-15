---
title: maspsx
description: Modern PSYQ Assembler
published: true
date: 2026-02-15T19:39:36.984Z
tags: 
editor: markdown
dateCreated: 2025-02-02T11:30:34.498Z
---

# maspsx

[maspsx](https://github.com/mkst/maspsx) or "Modern ASPSX" is a tool designed to mimic the behaviour of the PSYQ ASPSX assembler, allowing users to post-process the output from GCC and use modern GNU `as` to assemble an equivalent ELF object.


## Motivations

The vast majority of tooling around decomp (e.g. asm-differ, permuter, decomp.me) supports ELF objects. The SN Systems PSX toolchain ("PSYQ") uses a proprietary object format, and whilst there are some tools available that can convert between PSYQ -> ELF, the conversion is lossy and not all PSYQ functionality is supported.


## References

 - [psyq-obj-parser](https://github.com/grumpycoders/pcsx-redux/tree/main/tools/psyq-obj-parser)
 - [psyq2elf](https://gitlab.com/jype/psyq2elf)