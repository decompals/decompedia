---
title: Wii Menu
description: Wii Menu (Version 4.3) for Nintendo Wii
published: true
date: 2025-04-23T12:58:11.930Z
tags: 
editor: markdown
dateCreated: 2025-04-23T12:52:58.856Z
---

About
=====

A Work-in-Progress (WIP) 1:1 decompilation of the Wii Menu that started around April 2024.  

The project supports version 4.3 for all available regions: **USA**, **PAL**, **Japan** and **Korea**. Wii Mini and Virtual Wii (vWii) builds are not supported.  

Version 3.4 and 3.5 (Korea) share almost all of the symbols. Still there are some missing, such as the entire SD Card Menu, and a few symbols did change (mostly with extra arguments).  

The Wii Shop Channel shares the same code base and that too has symbols, such as having names for inline functions which the Wii Menu did not have.

History
=======

* **November 19, 2006**: The Wii console along with the Wii Menu was released.
* **June 21, 2010**: System Menu Version 4.3 was released, the last version.
* **March 2024**: Started the decompilation project.
* **October 19, 2024**: The build system now uses the [Decomp Toolkit](https://github.com/encounter/decomp-toolkit) tool.
* **Janurary 3, 2025**: The project structure moved on to use the [Decomp Toolkit Template](https://github.com/encounter/dtk-template).

Libraries
=========

First party
-----------

* **Revolution SDK**
  * The Revolution SDK seems to be recompiled on every release. For example the build date of 4.3 is `100420`, which translates to 2010/04/20, and the SDK build dates shows "Apr 20 2010".
  * It also looks to be a pretty early SDK revision.  
* **RevoEX**
* **NintendoWare for Revolution**
  * The version of NintendoWare for Revolution used for the Wii Menu would later be shipped in with the HOME Menu (HBM) library under the name `nw4hbm`, along with other small parts of Wii Menu code.
  * There are asserts in this version. However they are all just an `OSFatal` saying Error 004.
* **RVLFaceLib**
* **EGG**

Third party
-----------

* **TMCC JPEG**
* **eZiText** (for USA and PAL)
* **JustSystem ATOK** (for Japan)
* **GameSpy**
* **Opera Browser (WWW Library)**
  * This library is not in the executable file, but rather as a seperate RSO file.

External Resources
==================

* Decompilation projects that share a very similar SDK revision, such as [OGWS](https://github.com/kiwi515/ogws) and [Petari](https://github.com/SMGCommunity/Petari).
* Various games including a lot of information about SDKs from `.map` and `.elf` files
  * Tokyo Friend Park II - Ketteiban is the most useful one with it's huge amount of DWARF.
* [BSTool](https://github.com/koopthekoopa/BSTool) for handling the Wii Menu's exectuable.
* [WiiBrew](https://wiibrew.org/wiki/Main_Page) and [Yet Another GC Documentation](https://www.gc-forever.com/yagcd).

External Links
==============

* [Github Project Repository](https://github.com/koopthekoopa/wii-ipl)
* [Decompilation progress at decomp.dev](https://decomp.dev/koopthekoopa/wii-ipl)
