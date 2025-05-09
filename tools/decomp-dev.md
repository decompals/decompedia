---
title: decomp.dev
description: 
published: true
date: 2025-05-09T05:06:55.909Z
tags: 
editor: markdown
dateCreated: 2025-04-23T09:56:30.855Z
---

# decomp.dev

[decomp.dev](https://decomp.dev) is a hub for decompilation projects focusing on progress reporting.

Progress reports can be generated using [objdiff](https://github.com/encounter/objdiff), and the [protobuf schema](https://github.com/encounter/objdiff/blob/main/objdiff-core/protos/report.proto) is available to generate from your own tooling if desired.

The source code can be found [here](https://github.com/encounter/decomp.dev).

## API

decomp.dev has an [API browser](https://decomp.dev/api) that allows you to preview all project API endpoints, as well as generate and customize progress badge images.

## GitHub bot

Once your project is on decomp.dev, you can install the [GitHub bot](https://github.com/apps/decomp-dev) to the repository or organization. decomp.dev will automatically comment on PRs detailing any changes. PR comments may be configured in the [management interface](https://decomp.dev/manage).

The GitHub bot allows decomp.dev to receive notifications when a GitHub Actions workflow completes, rather than polling every 5 minutes.

## Integration guide

This guide focuses on generating progress reports using objdiff in your own build system. If you're setting up a GC/Wii decomp using [dtk-template](https://github.com/encounter/dtk-template), objdiff is already integrated, and these steps aren't necessary.

### 1. Build target objects

objdiff needs "target" objects that represent the **matched** state of the binary. Since most decompilation projects re-link a matching binary, you can think of the target objects as the set of objects passed to the linker for a matching build.

splat normally emits one `.s` file per function, but objdiff needs full objects to compare against.
One way to achieve this is to enable splat's `make_full_disasm_for_code` flag:

* CLI: `--make-full-disasm-for-code`
* YAML: `make_full_disasm_for_code: True`

This flag was introduced in [splat#448](<https://github.com/ethteck/splat/pull/448>).

When enabled, splat emits a full `.s` file for each `c`/`cpp` segment, that, once assembled, can be used as a "target" object for objdiff.

### 2. Build base objects

objdiff needs "base" objects that are **built from your source code**. Any units that don't have a source file yet may have a `null` base. objdiff will compare this set of objects to the target objects to calculate the progress information.

For projects that use `INLINE_ASM`, this adds *non-decompiled* functions to the objects, which is good for matching, but bad for progress calculation! 

One approach is to build the sources with `INLINE_ASM` disabled. Some projects configure a define like `-DSKIP_ASM=1` for this purpose, which [defines `INLINE_ASM` to nothing](<https://github.com/ladysilverberg/xenogears-decomp/blob/ae2a8ffce15f59fc9814286b7c322e99dc9d8ef7/include/include_asm.h#L4>). These non-matching base objects could be built at a **separate path** than the matching objects by your build system, specifically for use with objdiff.

**Note**: objdiff v3.0.0-beta attempts to automatically strip out functions that include a `.NON_MATCHING` label (emitted by the `INLINE_ASM` macro), so you *may* have success using your source-built objects without modification. Let me know!

### 3. Generate `objdiff.json`

An `objdiff.json` should be dynamically generated (e.g. by a Python script) at the root of the project.

`units` should contain **every object** that gets linked into the binary, so that objdiff has a **complete** view of the project
for progress calculation purposes.

If a file doesn't have a source file yet, it can have a `null` value for `base_path`.

(For more details, see the [objdiff README](<https://github.com/encounter/objdiff?tab=readme-ov-file#configuration>) and the [config.schema.json](<https://github.com/encounter/objdiff/blob/main/config.schema.json>))

`objdiff.json` example:

```json
{
  "$schema": "https://raw.githubusercontent.com/encounter/objdiff/main/config.schema.json",
  "custom_make": "make",
  "build_target": true,
  "build_base": true,
  "units": [
    {
      "name": "main/MetroTRK/mslsupp",
      "target_path": "build/asm/MetroTRK/mslsupp.s.o",
      "base_path": "build/src/MetroTRK/mslsupp.c.o",
      "metadata": {
        "progress_categories": ["sdk"]
      }
    }
  ],
  "progress_categories": [
    {
      "id": "game",
      "name": "Game",
    },
    {
      "id": "sdk",
      "name": "SDK",
    }
  ]
}
```

`build_target` and `build_base` are optional, but should be enabled if your build system is set up properly.

As in, if you can run `make build/asm/MetroTRK/mslsupp.s.o` successfully, then you should set `build_target` to `true`.
Otherwise, set it to `false`.

objdiff will execute the make command(s) when a project file changes and reload the objects.

### 4. Generate a progress report

Inside a make/ninja rule, or even directly in your CI workflow:

```
objdiff-cli report generate -o build/report.json
```

The `objdiff-cli` binary can be downloaded from the [GitHub releases](<https://github.com/encounter/objdiff/releases>).

If you'd like to fetch `objdiff-cli` automatically, check out [download_tool.py](<https://github.com/encounter/dtk-template/blob/main/tools/download_tool.py>) from GC/Wii's dtk-template, as well as how it's [integrated into the ninja build](<https://github.com/encounter/dtk-template/blob/2e907657cca5cd96ebad7ed1ce7a38c55a7b118b/tools/project.py#L512-L521>).

### 5. Publish the report in GitHub Actions

To automatically generate progress reports, you will need a GitHub Actions workflow that builds the project and generates the progres report.

The progress report should be uploaded as an artifact named `VERSION_report` (e.g. `SLUS_203.88_report`).

Projects that build multiple versions of the game can use a matrix strategy to build each version in parallel.

```yaml
- name: Upload progress report
  uses: actions/upload-artifact@v4
  with:
    name: ${{ matrix.version }}_report # e.g. SLUS_203.88_report
    path: build/report.json
```

### 6. Add the project to decomp.dev

Once progress reports are being uploaded on the default branch, if you're a repository admin, you can visit <https://decomp.dev/manage/new> and add the project. 🎉
