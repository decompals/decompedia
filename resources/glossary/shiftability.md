---
title: Shiftability
description: What shiftability is, why it is useful, and how to achieve it
published: true
date: 2025-05-07T03:48:46.792Z
tags: shiftability, glossary
editor: markdown
dateCreated: 2025-02-22T04:09:22.098Z
---



<!-- LOL i am not a good writer >.< --> 
<!-- i am NOT an expert in this, i am a NOVICE --> 

> WIP -- this is an unreviewed DRAFT. Information may be incomplete and/or misleading. This article only represents the understanding of one individual.
{.is-warning}



# Shiftability

  ## <!-- Bad --> Definition
  Shiftability is the ability to safely shift code around an executable without changing the behavior of the code at all. Shiftability allows changing the order of functions, adding or removing functions, and adding or removing TUs (source files). It is useful for modifying applications, i.e. developing mods for video games.
  ## Background
  Programming console games is often done in C/C++, using lots of pointers. Usually, these pointers are relative to other pointers, e.g. `auto *pFoo = &bar->mFoo` This forms like a tree of pointers, where the root of the tree is of interest. The root is not relative to another pointer, instead it is absolute. It could come from malloc, which does not affect shiftability. <!-- (TODO: maybe add code exmaples for these.) --> Or, it could come from a static variable. Static variables _do_ affect shiftability, causing problems: they have a fixed, static location in memory. Therefore, they have a fixed, constant address. <!-- (TODO: is this true? what are other case of hardcoded pointers) --> As a result, a program will use hardcoded pointer literals (e.g. `0x80001234`) to reference these static variables. 
  ## Problem
  Problems arise when more code/data is inserted or removed before the static variable. This changes the location of the static variable; it _shifts_ the variable and its data. That invalidates the hardcoded pointers. Being hard-coded, static, unchanging constants, hardcoded pointers will still point to the old location.
  ### What's supposed to happen
  Compilers generate object files, more precisely called "relocatable" object files. You may recognize them from their typical `.o` file extension. As their name suggests, they can be relocated, or put another way, *shifted*. When compiling object files, references to static variables will become **relocation entries** (sometimes called "relocations"). <!-- (TODO: is this true? what is it actually called) --> If one examines an `.o` file  using a tool like `objdump`, they will <!-- (TODO: will they tho? confirm accuracy/veracity. what is a compiler/platform agnostic way to phrase this?) --> see things like `RELOC_PPC32`; these are relocation entries. <!-- (TODO: example. maybe, use vabolds dtk training framework to make an example?) --> When the linker combines the `.o` object files into an `.elf` executable file, the linker will perform **relocation resolution**. The linker will pick a specific address to store static data, and use that address to replace all the associated relocation entries. So the final `.elf` executable is **not relocatable**.<!-- See [reference here](TODO: find a reference that doens't suck and actually makes some sense). -->
  ### What happens
  The first step of decompiling is splitting<!-- , see [splitting](TODO maybe link the melee page, it has a good explanatin. maybe split that section to its own article??) -->. Splitting undoes the linker's work. It converts ane executable program back into object files. However, undoing relocation resolution isn't straightforward. Short of fully decompiling and matching the entire prgram, there is no way to 100% guarantee that we have properly undone the relocation resolution step. In other words, shiftability cannot be proven without 100% `OK`. <!-- (TODO: is this true?) --> This is because relocation resolution is a lossy process. It erases information about the original data.  If a program's code has:
  ```cpp
  const auto foo = 0x80001234;
  static auto bar = "some data";
  printf("%d", foo);
  printf(bar);
  ```
  then looking at the final executable, how do we know `foo` is just a literal value and does not come from a relocation entry, but `&bar` *does* come from a relocation entry? <!-- TODO: I don't think this example is good or correct. --> We don't! And thus, problems occur.

  ## Fixing
  To fix, one must undo the relocation converting hardcoded pointers back to relocations. So first: find hardcoded pointers, and second: give them to your splitter so it can turn them back into relocation entries.
  ### when using [encounter/dtk](https://github.com/encounter/decomp-toolkit)
  encounter's decomp-toolkit for GC/Wii includes a splitter that tries to automatically detect relocations and undo relocation resolution. Thus, it tries to achieve perfect shiftablity automatically. However, sometimes it misses some relocations. Sometimes it fakes relocations, erroneously converting normal data into relocations. So, we have to use dtk's `noreloc` attribute to tell it not to treat normal data as a hardcoded pointer. <!-- (TODO: mayb add example of a string or somethign that looks liek a pointer. or jsut grep "noreloc" accross dtk projects to find examples.) --> See the tool's documentation for further details.