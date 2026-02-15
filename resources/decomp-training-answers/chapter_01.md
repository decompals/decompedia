---
title: Chapter 1 answers
description: 
published: true
date: 2026-02-15T19:40:08.532Z
tags: 
editor: markdown
dateCreated: 2025-05-17T21:24:20.865Z
---

# Chapter 1 answers

<details>
  <summary>Hint for weird_func, call_weird_func, and call_weird_func_2</summary>
  
Note that `r3` gets read from in `stw r3, 0x0(r31)` immediately following `bl weird_func`. What does this tell you about `weird_func`? 
  
</details>

<details>
  <summary>Solution and explanation of weird_func, call_weird_func, and call_weird_func_2</summary>
  
Even though `weird_func` appears to be empty, it's actually implicitly returning its first argument at `r3`, which gets written to the pointer `int *b`. Since `r3` is used by the ABI as both the first argument and the return register, no discrete move actually has to happen inside of `weird_func` and the compiler can simply write a single `blr`. 

```c
int weird_func(int a) {
    return a;
}

void call_weird_func(int a, int *b) {
    *b = weird_func(a);
}
  
void call_weird_func_2(int a, int *b) {
    a = a + 2;
    *b = weird_func(a);
}

```

 You're probably wondering why the initial definition of `int weird_func(void) { }` even compiles in the first place when nothing is returned even though it has an `int` return value. This wouldn't fly if you were going to port this code to another languages, but unfortunately because of the C language's rules, it's actually considered valid and allows `weird_func` and `call_weird_func` to compile to the same thing regardless of whether `weird_func` is written as returning its first argument or not. 

One of the key principles of decomp, what I'm trying to get at at by having you do these problems, is that you should understand and be aware of what the tooling you're using *can* automate, versus what it *cannot*. The reason I provided the initial code like that is because it's what an automated decompiler like Ghidra or m2c would typically output, since they'd decompile these functions in isolation (remember that `weird_func` could be in a different TU). You likely felt confusion or frustration if you focused primarily on `call_weird_func_2` and assumed that the other functions *had* to be correct because they have a green 100% label in objdiff. These types of misunderstandings are perhaps the biggest hurdle to overcome when starting out in decomp, which is why it's important to identify and question any assumptions you may unconsciously be making and learn more about how things work when you get confused.
