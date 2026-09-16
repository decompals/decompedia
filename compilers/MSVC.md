---
title: MSVC
description: Microsoft Visual C++ (16-bit and 32-bit Windows compilers)
published: true
date: 2026-08-20T13:42:59.000Z
tags: compiler, msvc, visual-cpp, microsoft
editor: markdown
dateCreated: 2026-08-20T13:42:59.000Z
---

# MSVC

MSVC (Microsoft Visual C++, also released as "Microsoft C") is Microsoft's C/C++ compiler for Windows. The full command-line line from **VC 1.0 (1992)** through **VC 11.0 (Visual Studio 2012)** is preserved and usable for matching decompilation of Windows 3.x / 9x / NT-era software. Unlike the console compilers on this site, MSVC has no native Linux build, so it is run through an emulation layer: wine (or [wibo](/tools/wibo)) for the 32-bit line and DOSBox for the 16-bit line.

## Versions

The preserved line splits into two families with very different runtimes:

- **16-bit (VC 1.0, 1.5, 1.52)** — Windows 3.x compilers. `CL.EXE` is a [Phar Lap TNT DOS-extender](https://en.wikipedia.org/wiki/Phar_Lap_(company)) binary that runs headless under DOSBox and produces 16-bit OMF objects.
- **32-bit (VC 2.0–11.0)** — the first 32-bit compiler is VC 2.0 (1994); everything after produces COFF objects and PE executables and runs under wine / wibo.

| Version | Year | CL.EXE | Notes |
|---|---|---|---|
| VC 1.0 | 1992 | — | 16-bit, Phar Lap; WinWorld 3.5" floppy set |
| VC 1.5 | 1993 | — | 16-bit, Phar Lap; archive.org `en_vc152` |
| VC 1.52 | 1995 | — | 16-bit, Phar Lap; archive.org `en_vc152_202512` |
| VC 2.0 | 1994 | 9.00 | first 32-bit compiler |
| VC 4.0 | 1995 | 10.00.5270 | |
| VC 4.1 | 1996 | 10.10.6038 | |
| VC 4.2 | 1996 | 10.20 | |
| VC 5.0 | 1997 | 11.00.7022 | SP1–SP3 ship the same CL.EXE |
| VC 6.0 | 1998 | 12.00.8168 / 12.00.8804 | see service packs below |
| VC 7.0 | 2002 | 13.00.9466 | .NET 2002 |
| VC 7.1 | 2003 | 13.10.3077 | .NET 2003 |
| VC 8.0 | 2005 | 14.00.50727 | VS 2005 |
| VC 9.0 | 2008 | 15.00.21022 | VS 2008 |
| VC 10.0 | 2010 | 16.00.30319 | VS 2010 |
| VC 11.0 | 2012 | 17.00.50522 | VS 2012; newest preserved version |

### VC 6.0 service packs

The VC 6.0 compiler line is **12.00.8168 from RTM through SP3** and **12.00.8804 from SP4 onward** — SP1/SP2/SP3 changed no compiler binaries and no include headers (verified against the official SP2 disc payload), so only the CRT/MFC sources, libs and runtime DLLs differ. SP4/SP5/SP6 carry the same 8804 `cl.exe` (verified byte-identical to the official **Visual Studio 6 SP4 CD**). The standalone SP1 payload (`VSE600SP1.EXE`) is not preserved in any public archive, so the SP1 tree is a reconstruction (RTM + the SP1-fixed files from the cumulative SP2 payload).

## Running MSVC

The [rebrew-toolchains](https://github.com/maci0/rebrew-toolchains) project packages every preserved version and service pack as a self-contained docker image, `rebrew/msvc:<version>-<arch>` — the image wraps the runtime and the entrypoint is a `cl` wrapper, so a compile is just:

```bash
docker run --rm -v "$PWD":/work -w /work rebrew/msvc:6.0-win32 /c /O2 f.c   # → f.obj
```

The 32-bit images run the compiler through wine by default; setting `REBREW_RUNNER=wibo` swaps in the [wibo](/tools/wibo) loader for much faster startup on plain console compiles. The 16-bit images (VC 1.0/1.5/1.52) run `CL.EXE` under headless DOSBox. [decomp.me](https://decomp.me) hosts a subset of these compilers (msvc6.0/6.3/6.4/6.5/6.5pp/6.6/7.0) on its platform, running them under wibo.

The 16-bit and 32-bit compilers produce **different object formats** that matter for matching: 16-bit OMF (decoded with a dedicated OMF parser) vs 32-bit COFF. The two are not interchangeable — a 16-bit Windows 3.x target must be compiled with the 16-bit line (VC 1.x), not VC 4.2+.

## Identifying an MSVC compiled binary

MSVC binaries are the most reliably fingerprintable Windows binaries, and multiple independent signals pin the **exact** version and service pack:

- **Rich header** — MSVC linkers (6.0+) embed a Rich header recording the compiler front/back-end builds. Combined with the linker version this pins the exact compiler: e.g. linker 6.0 + C1 build 9782 = `12.00.9782` = VC 6.0 SP6. The VC 6.0 service packs are distinct C1 builds: **8168 (RTM), 8447 (SP3), 8966 (SP5), 9782 (SP6)**.
- **Linker version** — VC 2.0–4.2 linkers write no Rich header, so the optional-header linker version alone names the version: 2.50 → VC 2.0, 3.0 → 4.0, 3.10 → 4.1, 4.20 → 4.2 (a bare 2.x is ambiguous with MinGW).
- **CRT imports** — the `msvcpX.dll` / `msvcrX.dll` import (msvcp60/70/71/80/90/100) is a secondary binder.
- **PDB** — a sibling `.pdb` carries an `S_COMPILE3` record with the compiler version and, for MSVC, the exact compiler flags.
- **Codegen fingerprints** — even without headers, code shape identifies era, optimization and (roughly) version: `/O2` wrapper calls load-first (`mov eax,[esp+4]; push eax; add esp,N`) vs `/O1` push-[mem] (`push dword [esp+4]; pop ecx`); pre-6.0 compilers hoist a small loop-invariant constant into a callee-saved register and store via it (`mov ebx,imm32; mov [mem],ebx`) where VC 6.0 emits `mov [mem],imm32` directly; int3 alignment padding vs GNU nops.  The full per-version matrix below is verified by compiling two probe sources through every preserved `CL.EXE` at `/O1` and `/O2` (plus every VC 6.0 SP at `/O2`) and disassembling the objects — probe #1 covers division, loops, FP math, char ops, stack frames, switches; probe #2 adds 64-bit arithmetic, FP constants (1.0/2.0/0.5), min/max/clamp, division by 5/9/12/100/1000, fixed-size memcpy/memset, rotates and sparse switches.

### Codegen behavior by version

| Behavior | VC 2.0 | 4.x | 5.0 | 6.0 | 7.0 | 7.1 | 8.0 | 9.0 | 10.0 | 11.0 |
|---|---|---|---|---|---|---|---|---|---|---|
| Magic-number division | — | — | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `lea esp,[esp]` loop-alignment nops | — | — | — | — | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `rol` for rotates (`x<<n \| x>>(32-n)`) | — | — | — | — | — | — | ✓ | ✓ | ✓ | ✓ |
| Memory-operand `imul [esp+4]` in div magic | — | — | — | — | — | — | — | — | ✓ | ✓ |
| `cmov` for min/max/clamp | — | — | — | — | — | — | — | — | — | ✓ |
| SSE2 FPU by default | — | — | — | — | — | — | — | — | — | ✓ |
| rep movs/stos inlining | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| /GS security cookie (default) | — | — | — | — | — | — | ✓ | ✓ | ✓ | ✓ |

- **Integer division** — VC 2.0 and 4.x emit real `div`/`idiv` (`mov ecx,3; xor edx,edx; div ecx`, with `sub edx,edx` for the zero-extend).  VC 5.0+ replace constants with magic immediates: `x/3u` → `mov eax,0xAAAAAAAB; mul [esp+4]`, `x/10u` → `0xCCCCCCCD`, `x/100u` → `0x51EB851F`, signed `x/7` → `0x92492493`, `x/10` → `0x66666667`.  The **tail is the version discriminator** (verified across 8 unsigned and 5 signed divisors):
  - **VC 5.0/6.0 unsigned:** `mov eax,edx; shr eax,N` (`8b c2 c1 e8 N`) — the shift applies to the *low* register.
  - **VC 7.0+ unsigned:** `shr edx,N; mov eax,edx` (`c1 ea N 8b c2`) — shift first, then move.
  - **VC 5.0/6.0 signed:** the sign fix round-trips through ECX — `mov eax,edx; sar eax,N; mov ecx,eax; shr ecx,31; add eax,ecx` (`8b c8 c1 e9 1f 03 c1`).
  - **VC 7.0+ signed:** `sar edx,N; mov eax,edx; shr eax,31; add eax,edx` (`c1 fa N 8b c2 c1 e8 1f 03 c2`).
  - **VC 10.0+ signed:** the divisor is no longer pre-loaded — `imul dword ptr [esp+4]` (`f7 6c 24 04`) multiplies the stack operand directly (VC 5.0–9.0 do `mov ecx,[esp+4]; imul ecx`).
- **Padding** — VC 2.0–6.0 pad with `90`, the 3-byte `8d 74 26 00` (`lea esi,[esi]`) and 7-byte `8d a4 24 00 00 00 00` nops; VC 7.0+ additionally emit `8d 64 24 00` (`lea esp,[esp]`) *inside* function bodies to align loop heads — a clean "VC 7.0+ codegen" marker even when the linker/CRT era says 6.0 (same independence as the constant-caching signal above).
- **FP constant encoding** — the way `a+1.0` compiles splits the line four ways: VC 2.0/4.x use `fld1` (`d9 e8`) + `fadd [esp+4]`; **VC 5.0 emits `fsub qword [−1.0]` (`dc 25`) with the *negated* constant in `.rdata`** (its table holds −1.0/−3.0 — a 5.0-only quirk); VC 6.0–10.0 use `fadd qword [+1.0]` (`dc 05`); VC 11.0 computes in SSE2 (`movsd xmm0,[esp+4]; addsd xmm0,[const]; movsd [esp+4],xmm0`) and hands the result to x87 for the return.
- **FPU** — VC 2.0–10.0 use x87 exclusively (`fld qword [esp+4]` = `dd 44 24 04`); FP constants load from `.rdata` via `fmul dword ptr [const]`, not `fld1`/`fldz`.  VC 11.0 (VS2012) defaults x86 to SSE2 (`addsd`/`mulsd`/`divsd` = `f2 0f 58/59/5e`).
- **String ops** — all versions inline memset/memcpy-shaped loops as `rep stosd` (`f3 ab`) / `rep movsd` (`f3 a5`); GCC and Borland bcc32 do not rep-stos the same source.  Fixed-size copies: 2–16 B inline as mov pairs, 32–64 B as `rep movsd` — identical in every version (not a discriminator).
- **Stack probes** — frames > 4 KB: `mov eax,<size>; call __chkstk` (32-bit) / `mov ax,<size>; call __aNchkstk` (16-bit); MinGW instead calls `___chkstk_ms`.
- **Frame pointer** — omitted at `/O1`/`/O2` (args at `[esp+4]`); `push ebp; mov ebp,esp` (`55 8b ec`) means `/Od` or `/Oy-`.
- **/GS** — VC 8.0+ default-instruments buffer functions with `__security_cookie` / `__security_check_cookie` (the symbols survive in unstripped binaries and objects).
- **min/max/clamp** — `imax`/`imin`/`clamp` compile to branches in VC 2.0–10.0 and to `cmovg`/`cmovl` (`0f 4f`/`0f 4c`) in VC 11.0 — the only MSVC version that emits conditional moves for them.  `iabs` (`x<0?-x:x`) uses `test; jns; neg` in VC 5.0–9.0 and `cdq; xor; sub` (`99 33 c2 2b c2`) in VC 10.0+.
- **Proven non-discriminators** — the VC 6.0 SP levels (SP1–SP6) are codegen-identical for every tested function: the SPs differ only in the Rich-header C1 build (8168/8447/8966/9782), not in code shape.  VC 4.0/4.1/4.2 are byte-identical to each other and to VC 2.0.  64-bit mul/div/shift stay helper calls (`__allmul`/`__alldiv`/`__allshl`) in every version — never inlined.
- **16-bit line** — VC 1.x always frames with `push bp; mov bp,sp`, ends with `leave` (`c9`), divides via `mov bx,N; div bx`, and intersperses `fwait` (`9b`) in FPU code.
- **Detect It Easy** — `diec` carries per-version MSVC signatures and is the quickest first pass.

A common trap: the compiler and the CRT/linker eras are independent fingerprints, so a binary can show a VC 6.0-era CRT while containing 4.x/5.0-compiled translation units (mixed-version builds are real — see [Projects](#projects)).

## Known quirks

- **C1.DLL/C2.DLL must come from the same toolchain.** MSVC compiler components are loaded at runtime, and wine finds them through `WINEPATH`. If the wrong toolchain's bin dir stays in the search path, `cl.exe` silently loads the *other* compiler's C1/C2 DLLs and the codegen comes from the wrong version — a per-function override to another MSVC version then quietly produces the default version's code. The toolchain's own bin dirs must replace `WINEPATH` (and `INCLUDE`/`LIB`/`PATH`).
- **NE header offset** — in 16-bit Windows NE executables, MSVC's linker places the NE signature at `e_lfanew` = 0x400 (past a fixed stub), whereas Borland puts it at 0x40 — a quick way to separate the two 16-bit toolchains.
- **Determinism** — MSVC objects embed the source path and the COFF TimeDateStamp (build time); the timestamp is the only non-deterministic byte, and byte-reproducibility gates mask it.

## Where to get it

- **[archaic-msvc](https://github.com/archaic-msvc)** — GitHub org with one preservation repo per compiler, VC 2.0 through VC 10.0, including every preserved VC 6.0 SP level and VC 5.0's SPs. The repos carry the original binaries, not repacks.
- **[archaic-toolchains](https://github.com/archaic-toolchains)** — the gap repos archaic-msvc does not carry: the 16-bit line (VC 1.0/1.5/1.52), VC 6.0 SP1/SP2/SP4 with their `Bin/`, VC 9.0 SP1's 15.00.30729 compiler (extracted from the official VS2008 SP1 DVD — this was "the only real gap"), plus the non-MSVC C line (Borland, Watcom, Delphi — see the other compiler pages).
- **decomp.me releases** — `github.com/OmniBlade/decomp.me/releases/download/msvcwin9x/` publishes msvc6.0/6.3/6.4/6.5/6.5pp/6.6/7.0 tarballs (compile-only, no `Lib/`).
- **archive.org** — `en_vc152` / `en_vc152_202512` (VC 1.5/1.52 media); **WinWorld** hosts the VC 1.0 3.5" floppy set (SZDD payloads).

All pinned sources are sha256-verified at image build time, so a changed source fails loudly instead of silently producing a different compiler.

## Projects

Public *matching* decomp of PC software is still young and much of it happens privately (e.g. via decomp.me's scrub queue), so the known list is short:

- [decomp.me](https://decomp.me) — hosts the MSVC line as its win32 compilers.
- [rebrew](https://github.com/maci0/rebrew) — compiler-in-the-loop workbench; its documented targets include the **Europa 1400 server** (a mostly-VC6.0 binary with scattered 4.2/5.0-compiled files — the mixed-version case above) and **SkiFree** (16-bit, VC 1.52).
