---
title: Chapter 1 answers
description: 
published: true
date: 2025-05-17T19:06:45.033Z
tags: 
editor: markdown
dateCreated: 2025-05-13T23:39:01.946Z
---

# Chapter 1 answers

<details>
  <summary>Hint for weird_func and call_weird_func</summary>
  
Note that `r3` gets read from in `stw r3, 0x0(r31)` immediately following `bl weird_func`. What does this tell you about `weird_func`? 
  
</details>

<details>
  <summary>Solution and explanation of weird_func and call_weird_func</summary>
  
Even though `weird_func` appears to be empty, it's actually implicitly returning its first argument at `r3`, which gets written to the pointer `int *b`. Since `r3` is used by the ABI as both the first argument and the return register, no discrete move actually has to happen inside of `weird_func` and the compiler can simply write a single `blr`. You should have been able to deduce that the second argument `r4`, which gets moved to `r31`, is a pointer since it appears on the right side of a store instruction as `0x0(r31)`, covered in the `store` function in Chapter 0. 

An important concept that I haven't highlighted explicitly but you could have figured out by carefully thinking about how volatile registers work is that if you see *any* read from `r3` before a write to `r3` following a function call, that function *has* to be defined as returning something, else it is not consistent with the ABI.  

One of the key principles of decomp, what I'm trying to get at at by having you do these problems, is that you should understand and be aware of what the tooling you're using *can* automate, versus what it *cannot*. It would be pretty disorienting if you, for instance, ran a decompiler that didn't take the r3 load into consideration and simply outputted `void weird_func(void) { }` and assumed that the function *had* to be empty because it's what the decompiler outputted. These types of misunderstandings are perhaps the biggest hurdle to overcome when starting out in decomp, which is why it's important to learn about how everything works as best as you can.

```c
int weird_func(int a) {
    return a;
}

void call_weird_func(int a, int *b) {
    *b = weird_func(a);
}
```
  
</details>