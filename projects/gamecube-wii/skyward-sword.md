---
title: The Legend of Zelda: Skyward Sword
description: 
published: true
date: 2025-04-28T06:55:26.358Z
tags: 
editor: markdown
dateCreated: 2024-11-04T15:52:31.951Z
---

## About the project

An in-progress decompilation of The Legend of Zelda: Skyward Sword for the [Nintendo Wii](/platforms/gamecube-wii). Currently builds NTSC-1.0 (SOUE01 Rev 0), with other versions planned. Skyward Sword HD for the Nintendo Switch is out of scope, though the Skyward Sword HD Randomizer community has done some reverse engineering.

The repository is hosted in the [ZeldaRET GitHub organization](https://github.com/zeldaret/ss), and progress can be viewed on [decomp.dev](https://decomp.dev/zeldaret/ss).

There is no known debug information for any Skyward Sword version, so a large part of this project requires original research. However, some foundational libraries and even the core engine can be found in other games with more debugging info available.

## History

* November 2011: Skyward Sword releases for the Nintendo Wii.
* ????: speedrunning community starts analyzing and reversing the game.
* May 2020: Work begins on sslib, a library for parsing and writing Skyward Sword's assets, later developed into the [Skyward Sword Randomizer](https://github.com/ssrando/ssrando) project.
* October 2020: The shared Ghidra repository is created in the ssrando community, previously research was done independently.
* July 2023: Work towards preparing for a Decomp is done via [ppcdis](https://github.com/SeekyCt/ppcdis), manually splitting asm files and finding code boundaries
* August 2023: The decomp project respository is created using [Decomp Toolkit](https://github.com/encounter/decomp-toolkit) and some files are being decompiled.
* June 2024: The project is moved into the ZeldaRET organization, and the Ghidra repository alongside it.

## Libraries

* [EGG](/libraries/egg)
* [NintendoWare for Revolution](/libraries/nw4r)
* [Revolution SDK](/libraries/dolphin-sdk)
* LibMS
* Various smaller libraries shared with [NSMBW](/projects/new-super-mario-bros-wii) and AC: City Folk:
  * fLib | core engine framework
  * cLib | some math stuff, random numbers
  * sLib | interpolation, easing, state management
  * mLib | wrappers for NW4R's g3d and lyt, wrappers for EGG, math types
* JSystem (JStudio, JParticle)
* Code borrowed from other games, e.g. Collision code

## External Resources

As mentioned above, much of our knowledge about Skyward Sword's code comes from other games:

* [New Super Mario Bros. Wii](/projects/new-super-mario-bros-wii) shares much of the core engine and libraries
  * Many [symbol names](https://rootcubed.dev/nsmbw-symbols/symbolList/) have been discovered - always worth checking
  * Also has a semi-active [decomp project](https://github.com/NSMBW-Community/NSMBW-Decomp)
* [Wii Sports/OGWS](/projects/wii-sports) has symbols via Wii Fit U ELF and various debug assertions
  * Active [decomp project](https://github.com/kiwi515/ogws)
* Big Brain Academy has a linker map file
  * Semi-active [decomp project](https://github.com/vabold/bba-wd)
* Tokyo Friend Park II - Ketteiban has full DWARF debug info
* [The Wind Waker](/projects/gamecube-wii/wind-waker) and [Twilight Princess](/projects/gamecube-wii/twilight-princess) are earlier entries in the series for similar hardware
  * Collision code is lifted straight from TP
  * JSystem code is very similar to TP's JSystem
  * Various graphics code resembles code found in TWW/TP (e.g. shadows)
  * Code found in cLib, mLib, sLib can sometimes be found in TWW/TP SComponent or mLib
  * EGG is a JSystem successor

Other information has been gathered through plain static and dynamic analysis.
In particular, the Skyward Sword Randomizer community has analyzed various file formats and game behaviors:

* [Skyward Sword Randomizer](https://github.com/ssrando/ssrando)
* [Skyward Sword Tools](https://github.com/lepelog/skywardsword-tools) for more file extraction and parsing
* [Skyward Sword Object Map](https://lepelog.github.io/ss-object-map/)

## How to decompile

For first-time setup, please refer to the GitHub project's [README](https://github.com/zeldaret/ss/blob/main/README.md). Some details that will be relevant after the initial setup are discussed in this section.

### Lack of symbols

Many functions will have a name like `fn_8002ECD0`. This is simply an automatically generated name based on the fact that it's found at virtual memory address `8002ECD0`. Since the function could be from code that is only found in Skyward Sword, we might not have an "official name" and you can choose any name for it that makes sense and follows the surrounding coding style.

When you write code to match this function, objdiff won't know that your function is related to `fn_8002ECD0` and won't compare them. Simply clicking on `fn_8002ECD0` will allow you to choose a function implemented by you and temporarily "map" these two functions.

After you have a function matched and you're reasonably certain about argument types, you'll have to make the name known to the rest of the tooling by updating `config/SOUE01/symbols.txt` (or the relevant REL `symbols.txt`). The most convenient way to do this is using the command line, which will apply all objdiff mappings in bulk:

```sh
python ./tools/custom/apply_objdiff_mappings.py
```

You can also manually change symbols entries one by one by right-clicking your symbol, choosing `Copy <mangled name>`, where "mangled name" is the least readable version of the function name, and then opening the `config/SOUE01/symbols.txt` file to replace `fn_8002ECD0` with the mangled name.

### Examples

Here are some example Pull Requests that are good self-contained examples for fully matching a file.

* [#82 d_lyt_sky_gauge OK](https://github.com/zeldaret/ss/pull/82) | a main DOL file with the sLib state system
* [#67 d_t_tdb OK](https://github.com/zeldaret/ss/pull/67) | a small REL (relocatable object file, dynamically loaded code)