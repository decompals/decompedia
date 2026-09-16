---
title: wibo
description: Minimal Win32 loader for running Windows compilers on Linux/macOS
published: true
date: 2026-08-20T13:42:59.000Z
tags: tool, wibo, wine, windows, loader
editor: markdown
dateCreated: 2026-08-20T13:42:59.000Z
---

# wibo

[wibo](https://github.com/decompals/wibo) is a minimal Win32 loader that runs simple command-line 32-bit Windows binaries on Linux and macOS. It was developed to run Windows compilers **faster than [Wine](https://www.winehq.org/)** — [decomp.me](https://decomp.me) uses it to run MSVC and other win32 compilers on its platform — and it is the modern lightweight alternative to a full wine setup for console tools like `cl.exe` or `bcc32.exe`.

## Why wibo instead of wine

Wine is a full Windows runtime: every run pays for wine server startup and prefix initialization. wibo skips all of that — it loads the PE image, resolves imports against its own built-in WinAPI shims, and runs the program directly, which makes compiler invocations roughly an order of magnitude faster to start. For the plain command-line tools decompilation actually uses (a compiler that reads a file, writes an object, and exits), that speed comes at a bounded compatibility cost.

## How it works

- Loads 32-bit PE binaries natively (the guest stays 32-bit even on 64-bit hosts; no multilib needed).
- Implements a subset of Win32 as shims, grouped by DLL in `dll/` (kernel32, ntdll, msvcrt, …), targeting pre-XP behavior — these are old compilers that don't expect modern WinAPI semantics.
- Embeds custom Wine CRT DLL builds from [encounter/winedll](https://github.com/encounter/winedll) (separate msvcrt/ucrtbase builds per ABI version, aliased where ABI-compatible: 70→71, 80→90, 110→120).
- Missing imports are **not** a hard failure: unknown imports resolve to stub pointers, and wibo only aborts if the program actually *calls* one.

## Usage

```
wibo [options] <program.exe> [arguments...]
```

```bash
wibo path/to/test.exe a b c
wibo -C path/to test.exe a b c          # chdir before launching
wibo --cmdline 'test.exe a b c' test.exe  # exact guest command line
```

`-D` (or `WIBO_DEBUG=1`) enables loader tracing — the usual first step when a binary misbehaves. `wibo path` converts between host and Windows-style paths.

## Limitations

wibo implements a subset of Win32, and which tools work is empirical. A concrete example: `bcc32` (Borland C++ 5.5) calls `GetThreadLocale` from kernel32 during startup, a shim wibo does not implement, so it aborts with `wibo: call reached missing import GetThreadLocale from kernel32` before compiling anything — while the MSVC `cl.exe` line runs fine. When a tool misbehaves, fall back to wine; the shim gap may also be implementable upstream (the repo documents its shim conventions and testing workflow).

## Getting wibo

- **GitHub releases** — static binaries: `wibo-i686` (Linux x86), `wibo-x86_64` (Linux x86_64), `wibo-macos` (macOS x86_64, Rosetta 2 supported).
- **From source** — `cmake --preset release && cmake --build --preset release` (self-checking WinAPI fixtures run via `ctest`).

## Integration

- [rebrew-toolchains](https://github.com/maci0/rebrew-toolchains) docker images bake in the static x86_64 build; the compiler wrappers select it with `REBREW_RUNNER=wibo` (default `wine`), sha256-pinned from the release.
- [decomp.me](https://decomp.me) runs its Windows compilers under wibo.

## Related

- [taviso/loadlibrary](https://github.com/taviso/loadlibrary) — initial inspiration.
- [retrowin32](https://github.com/evmar/retrowin32) — a similar project with different goals and architecture.

wibo is MIT licensed; the embedded Wine CRT DLLs are LGPLv2.1+ (see the repo's `winedll/` directory).
