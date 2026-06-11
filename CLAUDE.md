# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

`ffmpeg-stable-autobuild` is a GitHub Actions workflow that automatically builds **Windows x86/x64** FFmpeg binaries (nonfree, with libfdk_aac & DeckLink). It cross-compiles FFmpeg on Linux runners using the `ffmpeg-windows-build-helpers` script, then publishes releases.

The primary artifact is a single workflow file: `.github/workflows/build-ffmpeg.yaml`.

## Workflow architecture

Four sequential jobs, all gated by a version check:

1. **`check`** — decides whether to build
   - Fetches FFmpeg latest stable version from `endoflife.date/api/ffmpeg.json` (strips trailing `.0` for git tag compat)
   - Queries `ffmpeg-windows-build-helpers` latest master commit SHA via GitHub API
   - Retrieves this repo's latest release tag to compare
   - Outputs `run_build=true/false` (no longer crashes with exit code 1 on no-change)
   - Generates timestamp once in this job to avoid matrix race conditions
2. **`build`** — compilation (runs on `ubuntu-22.04`, matrix: `[win64, win32]`)
   - Runs only when `run_build == 'true'` or `force_build` is set
   - Checks out `ffmpeg-windows-build-helpers`, applies patches, compiles FFmpeg
   - Uploads `ffmpeg.exe`, `ffprobe.exe`, `ffplay.exe` as artifacts
3. **`archive`** — packages binaries into 7z
4. **`release`** — creates GitHub Release

## Key external dependency: ffmpeg-windows-build-helpers

The build script lives in `rdp/ffmpeg-windows-build-helpers`. When troubleshooting, **clone it and inspect**:

```bash
git clone https://github.com/rdp/ffmpeg-windows-build-helpers.git --depth 1 /tmp/ffmpeg-windows-build-helpers
```

The upstream owner is configurable via Actions variable `FWBH_REPOSITORY_OWNER` (defaults to `rdp` via `||` fallback).

Key files when debugging:
- `cross_compile_ffmpeg.sh` — main build orchestration
- `patches/mingw-w64-build-r22.local` — GCC cross-compiler bootstrap (tool downloads, system detection)

## Build-time patches applied in workflow

The `build` job applies three inline patches before compiling:

| Patch | Problem | Fix |
|-------|---------|-----|
| AOMedia toolchain path | libaom moved cmake toolchains from `build/cmake/toolchains/` to `cmake/toolchains/` | `sed` in `cross_compile_ffmpeg.sh` |
| Savannah GNU `config.guess` | `savannah.gnu.org/gitweb` is unreliable (502 errors) | Delete wget line, replace `sh config.guess` with `gcc -dumpmachine` |
| AMF `timeapi.h` | MinGW lacks `timeapi.h`, causing AMF encoder compile crash | Create stub header redirecting to `mmsystem.h`, pass via `--cflags=-I...` |

APT dependencies include `gcc-multilib` and `g++-multilib` to fix 32-bit CUDA cross-compilation.

## Platform assumptions (Windows-only)

- **Host**: always `x86_64-pc-linux-gnu` (GitHub Actions Ubuntu runner)
- **Target**: always Windows — `win64` (x86_64) or `win32` (i686) via `--compiler-flavors`
- Platform detection that varies per OS can be replaced with hardcoded defaults (e.g. `gcc -dumpmachine`)
- Issues about other architectures (ARM, macOS, Linux targets) do not apply

## Build triggers

| Trigger | Behavior |
|---------|----------|
| `schedule` (cron: `7 11 * * 0`) | Weekly Sunday check |
| `workflow_dispatch` | Manual, supports `force_build` (skip version check) and `no_release` (skip release) |
| `watch` (star) | On star event |

## Fork-specific considerations

- Forks do **not** inherit Actions variables. The workflow uses `${{ vars.FWBH_REPOSITORY_OWNER \|\| 'rdp' }}` as fallback.
- Fresh forks have no releases — the `check` job handles `gh api .../releases/latest` 404 gracefully with `if ! ... then` instead of letting JSON error strings pollute bash variables.
- `actions/attest` steps were removed; `archive` job only needs `contents: read`, `release` job needs `contents: write`.
- Without `force_build`, builds only trigger when FFmpeg version or helper commit SHA differs from the last release.

## Release format

- Tags: `YYYY-MM-DD-HH-MM-abcdefg-nX.Y.Z`
- Artifacts: `ffmpeg-nX.Y.Z-abcdefg-{win64,win32}-nonfree.7z`
- Contains: `ffmpeg.exe`, `ffprobe.exe`, `ffplay.exe`, nonfree `LICENSE`
