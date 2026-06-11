# CLAUDE.md

## Project overview

`ffmpeg-stable-autobuild` is a GitHub Actions workflow that automatically builds **Windows x86/x64** FFmpeg binaries (nonfree, with libfdk_aac & DeckLink). It cross-compiles FFmpeg on Linux runners using the `ffmpeg-windows-build-helpers` script, then publishes releases.

The primary artifact is a single workflow file: `.github/workflows/build-ffmpeg.yaml`.

## Workflow architecture (`build-ffmpeg.yaml`)

Four sequential jobs:

1. **`check`** — polls external APIs to decide whether a new build is needed
   - Fetches FFmpeg latest stable version from `endoflife.date/api/ffmpeg.json`
   - Queries `ffmpeg-windows-build-helpers` latest master commit SHA via GitHub API
   - Retrieves the latest release tag from this repo to compare versions
   - If no newer version/commit AND `force_build` is false → exit early

2. **`build`** — the actual compilation (runs on `ubuntu-22.04`, matrix: `[win64, win32]`)
   - Checks out `ffmpeg-windows-build-helpers` at the detected commit
   - Patches `patches/mingw-w64-build-r22.local` to avoid Savannah GNU dependency (see below)
   - Installs APT deps: ragel cvs yasm pax nasm gperf autogen autoconf-archive
   - Installs pip deps: meson ninja
   - Runs `./cross_compile_ffmpeg.sh` with `--compiler-flavors=${{ matrix.os }}`
   - Uploads `ffmpeg.exe`, `ffprobe.exe`, `ffplay.exe` as artifacts

3. **`archive`** — packages binaries into 7z, generates attestations
   - Downloads per-platform binaries
   - Appends nonfree LICENSE notice
   - Creates `.7z` archive and uploads with zero compression

4. **`release`** — creates GitHub Release from the archives
   - Runs only when `no_release` input is false (default)

## Key external dependency

### ffmpeg-windows-build-helpers (`rdp/ffmpeg-windows-build-helpers`)

The build script lives in a separate repo. When troubleshooting build failures, **clone it and inspect the scripts directly**:

```bash
git clone https://github.com/rdp/ffmpeg-windows-build-helpers.git --depth 1 /tmp/ffmpeg-windows-build-helpers
```

Key file: `patches/mingw-w64-build-r22.local` — this is the GCC cross-compiler bootstrap script, where most build-time failures (tool downloads, system detection) originate.

The upstream repo owner is configurable via Actions variable `FWBH_REPOSITORY_OWNER` (defaults to `rdp`).

### Known issues fixed in this fork

1. **Savannah GNU dependency** (commit `8c8274c`): The GCC bootstrap script downloads `config.guess` from `savannah.gnu.org/gitweb`, which is an unreliable deprecated endpoint. Replaced with `gcc -dumpmachine` — same output (`x86_64-pc-linux-gnu`), no network call.

   The sed patch in the workflow:
   ```bash
   sed -i \
     -e '/wget.*savannah\.gnu\.org/d' \
     -e 's/system_type="\$(sh config\.guess)"/system_type="\$(gcc -dumpmachine)"/' \
     patches/mingw-w64-build-r22.local
   ```

2. **Missing `FWBH_REPOSITORY_OWNER` variable** (commit `ce38ebb`): Forks don't inherit actions variables. Added `|| 'rdp'` fallback in the workflow expression.

3. **No existing releases** (commit `9411dc1`): Fresh forks have no releases, so `gh api .../releases/latest` returns 404. Added `|| tag_name=""` to handle gracefully.

## Platform assumptions (Windows-only)

This project **only builds Windows x86_64 and i686 binaries**. It does not target Linux, macOS, ARM, or any other platform.

When diagnosing build issues:
- The host is **always** `x86_64-pc-linux-gnu` (GitHub Actions Ubuntu runner)
- The target is **always** Windows (`win64` or `win32` via `--compiler-flavors`)
- Platform detection that varies per OS (like `config.guess`) can be replaced with hardcoded defaults — e.g., `gcc -dumpmachine` or `x86_64-pc-linux-gnu` directly
- Issues about other target architectures (ARM, macOS, etc.) do not apply and can be ignored

## Build triggers

| Trigger | Behavior |
|---------|----------|
| `schedule` (cron: `7 11 * * 0`) | Weekly Sunday check |
| `workflow_dispatch` | Manual trigger (supports `force_build` and `no_release`) |
| `watch` (star) | On star event |

## Version-checking logic

The `check` job compares:
- FFmpeg latest stable version (from endoflife.date) vs. last released version
- Latest `ffmpeg-windows-build-helpers` commit SHA (7 chars) vs. SHA in last release tag

If either differs, a new build proceeds. Tag format: `<datetime>-<7-char-sha>-<version>`.

## Release format

Release tags follow: `YYYY-MM-DD-HH-MM-abcdefg-nX.Y.Z`

Artifact names: `ffmpeg-nX.Y.Z-abcdefg-{win64,win32}-nonfree.7z`

Contains: `ffmpeg.exe`, `ffprobe.exe`, `ffplay.exe`, `LICENSE`.

## Common troubleshooting

### Build fails in `check` job
- Check `FWBH_REPOSITORY_OWNER` variable is set or fallback is working
- Check `endoflife.date/api/ffmpeg.json` is reachable
- Check repo has at least one release (or patch for empty state is in place)

### Build fails in `build` job
- Check Savannah GNU URLs — if any remain, replace with local alternatives
- Check `savannah.gnu.org` status — transient 502s are common
- Inspect the relevant script inside the cloned `ffmpeg-windows-build-helpers` repo
- System type detection can always use `gcc -dumpmachine` since host is always Linux x86_64

### Build fails in `archive` or `release` job
- Verify artifact names match between upload/download steps
- Check `actions/attest@v4` and other action versions are still available
