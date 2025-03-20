---
title: So you want to decompile an N64 game
description: a work-in-progress guide to N64 game decompilation
published: true
date: 2025-03-20T14:59:27.914Z
tags: 
editor: markdown
dateCreated: 2025-03-15T11:54:09.928Z
---

# So you want to decompile an N64 game...


This guide is a work-in-progress, it is incomplete, some liberties will be taken with the truth with the aim of keeping things simple and accessible.

## Intro

### What is a decomp

When we talk about game decompilation (or 'decomp') we usually mean a reverse-engineering of the game into a high-level language (typically C) which can be compiled back into a byte-perfect match of the original binary.

Writing C code that compiles back to perfect byte-code is a gargantuan task and will make up the majority of time and effort required in an decomp project, but there is also a large amount of reverse-engineering required - identifying what different areas of the ROM are, for example.

Before any C code can be written (and matched), the areas of code need to be identified, split out and disassembled.


## "Splitting" the ROM

An N64 ROM contains everything needed to run a game; the game code, the textures & models, music and more. In order to decompile the code, you'll need to find where it is and extract it.

[splat](https://github.com/ethteck/splat) is a tool designed to make splitting an N64 ROM into its constituent parts (code, images, models etc), it will disassemble code for you, it will spit out PNG images - and it's highly extensible, if you have custom data types in your ROM, you can write extensions to process them.

### Finding code sections

[findcode](https://github.com/decompals/findcode) is a tool that will scan an N64 ROM and spit out the offsets where it believes code to be.

### libultra

libultra is the collection of library functions provided by SGI to developers who wanted to write games for the N64. 

Most (all?) N64 retail games were stripped of debug information (including variable names) as part of the build process. [n64sym](https://shygoo.github.io/n64sym/web/) will scan your N64 ROM and emit symbol-address pairs where it believes libultra functions and variables exist in the ROM. Note that it may not get things 100% correct.

## Overlays

What is an overlay?

### Finding overlays

Look for calls to `osInvalICache` - the function that invalidates the instruction cache. This is called when new instructions are copied into memory. Functions that load overlays will mostly likely call `osInvalICache`. These functions will DMA ROM data into memory.






