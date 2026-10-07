---
title: ELF (Executable and Linkable Format)
description: Decomp-oriented information on the ELF format
published: false
date: 2026-10-07T16:45:13.295Z
tags: elf, binary, executable, matching
editor: markdown
dateCreated: 2026-10-07T16:45:13.295Z
---

# ELF (Executable and Linkable Format)

[ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) is a standard, widely-used executable format, used across both Unix-like and proprietary
platforms like consoles.

For basic information on the structure of an ELF file, see the [Wikipedia](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) page.
This page is more oriented towards how the ELF format affects decompilation.

## Platforms

This is a (incomplete) list of platforms which use ELF as their **primary**
executable type for software.

- [Playstation 2](/platforms/playstation-2)

Note that many decomp toolchains use ELF as an intermediate step
for their binary output.
GNU [binutils](https://en.wikipedia.org/wiki/GNU_Binutils) is often used to transform an output ELF into some arbitrary format that is either
- the actual binary format the platform used
- a format that is easier to match against
  - For example, it's common to `objcopy` an ELF's contents into a "rom" which
    only contains the code and data sections, removing all ELF metadata/headers.
    This removes the requirement of matching the original binary's metadata, and
    also allows for [shiftability](/resources/glossary/shiftability).
    
## Global Pointers

On platforms which support a "global pointer", data can be segmented into
"small" and "large" groups. This is typically done by passing the `-G[N]`
flag to the compiler, where `[N]` represents the maximum size (in bytes) of
"small" data.

This is an optimization; the global pointer allows these platforms to access
the small data with one `gp`-relative load, when an absolute load would
normally require two instructions.

When `-G` is non-zero, some sections will be split into two, with one prefixed
with `s` (e.g. `.data` vs. `.sdata`). The `s` indicates that the section stores
"small" data whose length in bytes is below the specified `-G` limit.

## Sections

An ELF binary is split into "sections" which each house some distinct component
of the binary (either code or data).

Many platforms have their own custom sections; these are documented later in
this article.

### `.text`

`.text` is the primary section for all executable code. All ASM and C functions
are typically placed in here.

It is possible for data to exist in the `.text` section. For example, some
compilers have options to place jump tables inside of `.text` next to their
respective functions.

### `.rodata`

As the name implies, `.rodata` is used for read-only data and variables.
Most commonly, this includes the following:

- jump tables from `switch` statements
- string literals
- `const` variables

```c
const int MY_VALUE = 5;

void foo() {
  // "Hello, world!" will be implicitly stuck into `.rodata`.
  printf("Hello, world!");
}

void bar(int baz) {
  // If the compiler generates a jump table of addresses/offsets,
  // the table will typically be stored in `.rodata`.
  // Note that some compilers can optimize short/simple `switch`
  // statements into `if`/`else` chains.
  switch (baz) {
    case 1:
      // ...
    case 2:
      // ...
    case 3:
      // ...
    default:
      // ...
  }
}
```

### `.data` and `.sdata`

These sections contain non-`const` variables defined with initial values.

```c
// Found in `.sdata` if `-G` is set to `-G4` or higher.
// Found in `.data` otherwise.
static int bar = 4;

// Found in `.sdata` if `-G` is set to `-G8` or higher.
// Found in `.data` otherwise.
uint64_t baz = 5ull;

u8 foo[10] = { 0xDE, 0xAD, 0xBE, 0xEF, 0, 0, 0, 0, 0, 0 };
```

### `.bss` and `.sbss`

Similarly to `.data` and `.sdata`, but this is used for *uninitialized* variables.

```c
// Found in `.sbss` if `-G` is set to `-G4` or higher.
// Found in `.bss` otherwise.
static int bar;

// Found in `.sbss` if `-G` is set to `-G8` or higher.
// Found in `.bss` otherwise.
uint64_t baz;

u8 foo[10];
```

## COMMON data



## Linking

Most linkers have deterministic behavior. When a linker is invoked to create
a final executable from multiple objects, it will produce each section
of the binary by deinterleaving the associated section from each source object.

For example, with three incoming objects, `foo.o`, `bar.o`, and `baz.o`,
and a final binary `wut.elf`:

```
foo.o       bar.o       baz.o
- .text     - .rodata   - .text
- .rodata   - .text     - .rodata
            - .data     - .data
            - .bss      - .sdata
                        - .sbss
                        - .bss
```

Assuming we link in the order `(foo, bar, baz)`, the final binary
might look like:
```
wut.elf
- .rodata
  - .rodata (foo)
  - .rodata (bar)
  - .rodata (baz)
- .text
  - .text (foo)
  - .text (bar)
  - .text (baz)
- .data
  - .data (bar)
  - .data (baz)
- .sdata
  - .sdata (baz)
- .sbss
  - .sbss (baz)
- .bss
  - .bss (bar)
  - .bss (baz)
```

Note that both the object order `(foo, bar, baz)` and the final section order
here are arbitrary and can be specified by the user/developer through a linker
script.
The default section order is typically platform-defined.

## Platform-Specific Information

### PSX

### PS2

#### `.vutext` and `.vudata`

These sections contain microcode/data for the PS2's vector units (VUs).

#### `.lit4` and `.lit8`

These sections store `float` and `double` literals (_not_ variables),
respectively.

Similar to `.rodata`, because these values are literals, matching code that uses
them may be difficult or impossible unless the actual literal is used. This
typically implies that all of the literals in a TU must be matched at the same
time.


