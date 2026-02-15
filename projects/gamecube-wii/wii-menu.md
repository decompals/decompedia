---
title: Wii Menu
description: Wii Menu (Version 4.3) for Nintendo Wii
published: true
date: 2026-02-15T19:40:05.114Z
tags: 
editor: markdown
dateCreated: 2025-04-23T12:52:58.856Z
---

About
=====

A Work-in-Progress (WIP) 1:1 decompilation of the Wii Menu that started around April 2024.  

The project supports version 4.3 for all available regions: **USA**, **PAL**, **Japan** and **Korea**. Wii Mini and Wii U (Virtual Wii/vWii) builds are not supported. However, there are plans for building an exectuable compatible with those consoles.  

Symbols are stored in the `main.sel` file which are for RSO Files. Since version 4.0, the amount of symbols reduced to a very small amount. However, version 3.4 and 3.5 (Korea) include almost all of the symbols. Still there are some missing and a few symbols did change (mostly with extra arguments).  

The Wii Shop Channel shares the same code base and that too has symbols in its `main.sel` file, such as having names for inline functions which the Wii Menu did not have.

History
=======

* **November 19, 2006**: The Wii console along with the Wii Menu was released.
* **June 21, 2010**: System Menu Version 4.3 was released, the latest version.
* **March 2024**: Started the decompilation project.
* **October 19, 2024**: The build system now uses the [Decomp Toolkit](https://github.com/encounter/decomp-toolkit) tool.
* **Janurary 3, 2025**: The project structure moved on to use the [Decomp Toolkit Template](https://github.com/encounter/dtk-template).
* **September 29, 2025**: 25% of 4.3U/E was decompiled.

Libraries
=========

First party
-----------

* **Revolution SDK** (RVL_SDK)
  * The Revolution SDK seems to be recompiled on every release. For example the build date of 4.3 is `100420`, which translates to 2010/04/20, and the SDK build dates shows "Apr 20 2010". The actual SDK date seems to be between November 2007 and March 2008.
* **Revolution Extension** (RevoEX)
* **NintendoWare for Revolution** (NW4R)
  * The version of NintendoWare for Revolution used for the Wii Menu would later be shipped in with the HOME Menu (HBM) library under the name `nw4hbm`, along with other small parts of Wii Menu code.
  * There are asserts in this version. However they are all just an `OSFatal` saying Error 004.
* **Revolution Face Library** (RVLFaceLib)
* **EGG**

Third party
-----------

* **TMCC JPEG**
* **eZiText** (for USA and PAL)
* **JustSystem ATOK** (for Japan)
* **GameSpy**
* **Opera Browser (WWW)**
  * This library is not in the executable file, but rather as a seperate RSO file.

External Resources
==================

* Decompilation projects that share a very similar SDK revision, such as [OGWS](https://github.com/kiwi515/ogws), [dolsdk2004](https://github.com/doldecomp/dolsdk2004) and [Petari](https://github.com/SMGCommunity/Petari).
* Various games including a lot of information about SDKs from `.map` and `.elf` files
  * Challenge Me -  Word (and Brain) Puzzles reveal to have AR files of SDK libraries shipped in dated December 12th 2009.
    * [Decompilation of those libraries](https://github.com/doldecomp/sdk_2009-12-11)
  * Tokyo Friend Park II - Ketteiban is the most useful one with it's huge amount of DWARF.
* [BSTool](https://github.com/koopthekoopa/BSTool) for handling the Wii Menu's exectuable.
* [WiiBrew](https://wiibrew.org/wiki/Main_Page) and [Yet Another GC Documentation](https://www.gc-forever.com/yagcd).

External Links
==============

* [Github Project Repository](https://github.com/koopthekoopa/wii-ipl)
* [Decompilation progress at decomp.dev](https://decomp.dev/koopthekoopa/wii-ipl)
