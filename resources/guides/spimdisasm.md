---
title: Getting Started with spimdisasm
description: 
published: true
date: 2026-02-15T19:40:27.738Z
tags: 
editor: markdown
dateCreated: 2025-07-30T21:32:58.681Z
---

# Getting Started With spimdisasm
spimdisasm is a disassembler for MIPS binaries. It started life as a tool for decompilations of Nintendo 64 games, but has since been extended to support other MIPS-based platforms, like the PlayStation and Playstation 2.

## Install Python

The minimum Python version you need is 3.9. You can get it from python.org. If you're still running Windows 7, you can get an unofficial version from https://github.com/adang1345/PythonWin7.

During installation, make sure that pip, the package manager, also gets installed.

## Install spimdisasm

Open a command line prompt, and, assuming python.exe is on your PATH, input the following command:

`python -m pip install -U spimdisasm`

## Basic command

Before invoking spimdisasm, you need to gather information about what you'll be disassembling, and know which options to use them on.

* singleFileDisasm: Most of the time, you'll want to disassemble a single file.
* --instr-category: cpu, rsp, r3000gte (PSX), r4000allegrex (PSP), or r5900 (PS2).
* --endian: little or big
* --vram: The starting address of the virtual memory space. Often, this is 0x80000000.
* --start: The virtual memory space starts with read-only data, so you need to specify the offset where the code starts. One way to find this is to open your program in Ghidra, go to the beginning of the code segment, and look at the Byte Source Offset (either in a tooltip or in a column).
* --end: You also need to specify where the code segment ends.

Once you have all this information, you just need to append the filename and the output folder. Here's an example:

`spimdisasm singleFileDisasm --instr-category r3000gte --endian little --start 0x1a90 --vram 0x80039290 --end 0x32134 SLPM_800.80 output`

## Symbols

Having a dissambly is nice, but having one with symbols is nicer. If you have debug symbols for your game, you can make a file with name-value pairs and feed it to spimdisasm, the format is described in more detail in [Splat's Wiki](https://github.com/ethteck/splat/wiki/Adding-Symbols).

Example entry: `main = 0x80039290`

Specify this file using the `--symbol-addrs` option, followed by the filename.
