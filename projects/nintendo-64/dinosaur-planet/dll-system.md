---
title: DLL System
description: Dinosaur Planet's custom DLL system.
published: true
date: 2025-01-20T01:42:25.079Z
tags: 
editor: markdown
dateCreated: 2025-01-20T01:40:03.918Z
---

# DLL System
Dinosaur Planet swaps code and data in and out of memory by use of its DLL system. Many different systems and entities are coded as separate DLLs so only the ones that are currently needed are loaded in memory. These exist in the game's filesystem in a [custom format](/projects/nintendo-64/dinosaur-planet/dll-system/dll-format).

## Reference Counting
Only one instance of a given DLL will be loaded at a time in memory. DLLs are ref counted to ensure a DLL can only be unloaded if no game code is currently using it. When game code needs access to a DLL, it will either: load the DLL if it's not in memory or increment the existing instance's ref count. When game code no longer needs access to a DLL, the ref count will be decremented and if the ref count reaches zero, the DLL will also be unloaded.

## Constructors/Destructors
Each DLL defines a constructor and destructor function. When a DLL is loaded the constructor is ran and when it is unloaded the destructor is ran. Incrementing/decrementing ref counts on a DLL will not run these functions. Game code that derives instances of objects from a DLL (such as the game's entity system) tracks state *outside* of the DLL and use separate functions for constructing and destroying those instances.

## Exports
DLLs provide a "public interface" in the form of its export table. This is a list of pointers to some of a DLL's functions that outside code can use to invoke DLL code. When a DLL is requested by game code, a reference to the DLL's export table is returned.
