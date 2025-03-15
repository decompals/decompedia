---
title: So you want to decompile an N64 game
description: a work-in-progress guide to N64 game decompilation
published: true
date: 2025-03-15T11:54:09.928Z
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

## The N64 ROM

An N64 ROM contains everything needed to run a game; the game code, the textures & models, music and more. 

< MORE TO COME! >