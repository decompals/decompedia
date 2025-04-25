---
title: Luigi's Mansion
description: Info about Luigi's Mansion decompilation for GameCube
published: true
date: 2025-04-25T13:12:28.585Z
tags: 
editor: markdown
dateCreated: 2025-04-23T09:05:15.836Z
---

| [Progress](https://decomp.dev/Moddimation/YasikiDolphin) | [Repository](https://github.com/Moddimation/YasikiDolphin) | [Discord](https://discord.gg/hKx3FJJgrV) |
|------------------|---------|----------|

An in-progress decompilation of [Luigi's Mansion](https://wikipedia.org/wiki/Luigi’s_Mansion) for the [Nintendo GameCube](https://wikipedia.org/wiki/Nintendo_GameCube).
The focus is currently laid on the Japanese release (GLMJ01), but configurations exist for all [available versions](#versions). The original creator first focused on the North American release (GLME01), and it's symbol map was put in a backup file.

There are no known symbol maps for this game, however, it does come with [RTTI](https://www.sandordargo.com/blog/2023/03/01/binary-sizes-and-rtti).

History
---

### Game
<ul>
  <li><span style="display:inline-block; width: 110px">Aug. 24 2000</span> Presented at SpaceWorld 2000 simply as a tech-demo to show the power of the gamecube</li>
  <li><span style="display:inline-block; width: 110px">???. ?? 2000</span> Nintendo decides to shape a full game Project out of that demo.</li>
  <li><span style="display:inline-block; width: 110px">Aug. 24 2001</span> Presented at SpaceWorld 2001 as an actual game title</li>
  <li><span style="display:inline-block; width: 110px">2001 - 2002</span> Various game releases on gamecube</li>
  <li><span style="display:inline-block; width: 110px">Sep. 13 2018</span> 3DS port announced</li>
  <li><span style="display:inline-block; width: 110px">Oct. 19 2018</span> 3DS port released</li>
</ul>

### Project
<ul>
  <li><span style="display:inline-block; width: 110px">Sep. 9 2022</span> luigis-mansion channel created in discord</li>
  <li><span style="display:inline-block; width: 110px"><b>Dec. 15 2022</b></span> Sage-of-Mirrors creates <a href="https://github.com/Sage-of-Mirrors/zmansion">zmansion repo on github</a></li>
  <li><span style="display:inline-block; width: 110px">Dec. 18 2023</span> Sage-of-Mirrors last activity in #luigis-mansion channel</li>
  <li><span style="display:inline-block; width: 110px"><b>Dec. 12 2024</b></span> CoNesTra forks <a href="https://github.com/CoNesTra/zmansion">Sage-of-Mirrors/zmansion</a> and updates toolchain</li>
  <li><span style="display:inline-block; width: 110px"><b>Mar. 20 2024</b></span> Moddimation forks <a href="https://github.com/Moddimation/zmansion">CoNesTra/zmansion</a> </li>
  <li><span style="display:inline-block; width: 110px">Apr. 9 2025</span> #luigis-mansion channel is revived</li>
  <li><span style="display:inline-block; width: 110px">Apr. 23 2025</span> project is back on decomp.dev</li>
</ul>


Versions
---
| Config ID | Region | Variant | Started? | Release Date |  Build Date  | SDK Rev. |  SDK Build   | Apploader Build |
|-----------|--------|---------|----------|--------------|--------------|----------|--------------|------------------|
| GLMJ01    | Japan  | Release |    Yes   | Sep. 14 2001 | Aug. 28 2001 |    37    | Jul. 19 2001 | Apr. 04 2001 |
| GLME01    |  USA   | Release |   [No]   | Nov. 18 2001 | Sep. 24 2001 |    37    | Jul. 19 2001 | Aug. 9 2001 |
| GLME01_1  |  USA   |  Demo   |    No    | Oct.    2001 | Sep. 28 2001 |    45    | Sep. 08 2001 | Sep. 08 2001 |
| GLMP01    | Europe |  Demo   |    No    | Mar.    2002 | Jan. 21 2002 |    49    | Dec. 17 2001 | Nov. 30 2001 |
| GLMP01_1  | Europe | Release |    No    | May   3 2002 | Dec. 17 2001 |    49    | Dec. 17 2001 | Nov. 30 2001 |
| GLMP01_2  | Europe | Release |    No    | May  17 2002 | Dec. 17 2001 |    49    | Dec. 17 2001 | Nov. 30 2001 |

Libraries
---
- Dolphin SDK
- JSystem
- Jaudio (C)

External resources
---
- [Luigi's Mansion modding wiki (LMHack)](https://www.lmhack.net/index.php/Main_Page)