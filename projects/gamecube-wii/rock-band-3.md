---
title: Rock Band 3
description: Info about the Rock Band 3 Decompilation
published: true
date: 2025-04-23T08:36:36.574Z
tags: 
editor: markdown
dateCreated: 2025-03-21T21:59:10.330Z
---

## About
This is a reverse engineering project aimed at producing a functionally equivalent decompilation of the Wii version of Rock Band 3. It currently focuses on decompiling the [September 1st, 2010 (Bank 8) prototype](https://hiddenpalace.org/Rock_Band_3_(Sep_1,_2010_prototype)) of the game.
The repository can be found [here](https://github.com/DarkRTA/rb3), and the progress can be tracked on [decomp.dev](https://decomp.dev/DarkRTA/rb3).


## History
* October 26th, 2010: Rock Band 3 is released.
* September 7th, 2021: Milohax releases Guitar Hero 2 Deluxe, further popularizing modding Harmonix games.
* January 9th, 2022: MiloHax begins work on Rock Band 3 Deluxe, a mod that adds quality of life features through the game's scripting system.
* May 4th, 2022: RB3Enhanced begins work on reverse engineering in order to add quality of life features.
* September 20th, 2023: Decompilation begins on the Wii version of Rock Band 3.
* February 6th, 2024: pstrick1 dumps and releases a [Rock Band 2 debug build](https://hiddenpalace.org/Rock_Band_2_(Oct_6,_2008_Wii_prototype)) from a RVT-R disc. He also provides a [listing to an RVT-H development Wii with Rock Band 3 running](https://web.archive.org/web/20240206171237/https://www.ebay.com/itm/134924286068). A group buy is setup to purchase the Wii.
* February 17th, 2024: The RVT-H Wii's contents are dumped and [released](https://hiddenpalace.org/MiloHax). There are 5 beta builds of Rock Band 3, all with some form of symbol information.
* February 18th, 2024: The decompilation is restarted with the goal of decompiling the September 1st, 2010 prototype found on the Wii.
* August 21st, 2024: Percentage tracking gets changed from linked code to fully matched functions. This boosts the percent from 6% to 17.50%.
* March 14th, 2025: MattKC releases a [video](https://youtu.be/WdJ-Hqx3rNw?si=HZjPJ02zrPDmsWGr) about the decompilation.
* March 18th, 2025: The decompilation reaches 50% fully matched functions.

## Libraries Used
* [Revolution SDK](/libraries/dolphin-sdk)
* [Bink](https://www.radgametools.com/bnkmain.htm)
* [zlib](https://github.com/madler/zlib)
* Quazal
* [STLport](https://github.com/karottc/STLport-5.2.1)
*	[Vorbis](https://github.com/xiph/vorbis)
* [Speex](https://gitlab.xiph.org/xiph/speex)
* [Json-C](https://github.com/json-c/json-c)


## Resources
* [Rock Band 2 Debug DWARF Info](https://github.com/DarkRTA/rb3/tree/master/doc/rb2_dump)
* [RB3Enhanced Source Code](https://github.com/RBEnhanced/RB3Enhanced)
* [DTA Scripts](https://github.com/hmxmilohax/milo-script-library) -  Rock Band 1 PS2 scripts are in their raw format, which includes developer comments.
* Games that shipped with .elf's with symbols.
	* Rock Band 2 (Wii)
	* Green Day: Rock Band (Wii)
* Various .map symbol files found in prototype builds.
	* [RB2 06/10/2008 Prototype](https://github.com/hmxmilohax/milo-executable-library/blob/main/rb2/Wii%20Prototype%20(Debug)/band_r_wii.map)
	* [RB3 19/01/2010 Prototype](https://github.com/hmxmilohax/milo-executable-library/blob/main/rb3/Wii%20Proto%20(Bank%206)%20(Debug)/band_r_wii.map)
	* [RB3 16/03/2010 Prototype](https://github.com/hmxmilohax/milo-executable-library/blob/main/rb3/Wii%20Proto%20(Bank%205)%20(Debug)/band_r_wii.map) 
	* [RB3 29/08/2010 Prototype](https://github.com/hmxmilohax/milo-executable-library/blob/main/rb3/Wii%20Proto%20(Bank%202)%20(Debug)/band_r_wii.map) 
	* [RB3 01/09/2010 Prototype](https://github.com/hmxmilohax/milo-executable-library/blob/main/rb3/Wii%20Proto%20(Bank%208)%20(Debug)/band_r_wii.map)
  * [Dance Central E3 Prototype](https://github.com/hmxmilohax/milo-executable-library/blob/main/dc1/E3%202010%20(Debug)/ham_r.map)
      


## Contributing
* [CONTRIBUTING.MD](https://github.com/DarkRTA/rb3/blob/master/CONTRIBUTING.md)

