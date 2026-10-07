---
title: ELF (Executable and Linkable Format)
description: Decomp-oriented information on the ELF format
published: true
date: 2026-10-07T17:36:34.349Z
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

If `-G` is disabled on these platforms (`-G0`), the "small" data sections cease
to exist and all data is folded into the main sections.

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

### `COMMON` and `.scommon`

These are relatively rare sections that are only enabled with compiler flags
(`-fcommon` in GCC).

Particularly in C code, a global variable might need to be accessible from
multiple different [translation units](https://en.wikipedia.org/wiki/Translation_unit_(programming)) (TUs). This is typically handled by
using `extern` with a forward declaration in a header:

```c
// explode.h
// Exported declaration so other TUs can use this variable.
extern int foo[2];

// explode.c
// Actual storage location of the variable.
int foo[2] = { 0, 1 };
```

However, if the variable storage isn't actually initialized, the C standard
considers it a "tentative definition"; there's no way to know whether it's
_actually_ stored in `explode.c`'s TU. Some other TU might declare its own
storage for `foo`.

```c
// explode.h
// Exported declaration so other TUs can use this variable.
extern int foo[2];

// explode.c
int foo[2];

// explosion.c
int foo[2]; // Problem; who actually owns the storage?
```

With `-fno-common`, this leads to a linker error; there are multiple definitions
of the same variable.

With `-fcommon`, the symbol is defined as `.comm`, and the linker actually
attempts to resolve this by:

- moving `foo` to a `COMMON` section, either inside of or next to `.bss`
- resolving all references to that variable to use the `COMMON` location

The two different definitions of `foo` now resolve to the same place.

Compared to normal `.bss` data, however, `COMMON` data has several notable
behavior quirks.

#### Section Location

Depending on the linker script, `COMMON` symbols for a TU can be
located in one of two places in the final executable:

- merged with the TU's `.bss` section
- merged together with all other `COMMON` symbols in the binary

The latter case is much easier to detect for binaries with debug symbols.
When splitting the binary, you will encounter TUs which seemingly have
multiple `.sbss` or `.bss` segments (normally impossible).
This is a hallmark of `COMMON` usage.

The former case is easier to work with, but can introduce potentially insidious
matching problems because of...

#### Variable Reordering

When the linker is resolving multiple definitions, it needs to keep track of
which variables it has seen before. This is typically done through a
hashing function; two variables with the same name will resolve to the
same location.

However, because the variable storage is now independent of a specific TU,
the linker will **arbitrarily reorder variables** with some
implementation-defined method.

Typically, the new variable order depends on the output of the hashing
function. In other words, **variable names will directly influence the linked data order.**

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

This deterministic behavior is what allows splitting tools,
like `splat` and `dtk`, to effectively replicate the original object files
of an executable.

Note that both the object order `(foo, bar, baz)` and the final section order
here are arbitrary and can be specified by the user/developer through a linker
script.
The default section order is typically platform-defined.

## Platform-Specific Information

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


