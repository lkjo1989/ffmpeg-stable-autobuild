# FFmpeg Stable Autobuilds for Windows

[FFmpeg](https://ffmpeg.org) stable nonfree release builds for Windows (x86_64 and i686), automatically cross-compiled on Linux via GitHub Actions.

[**Downloads**](https://github.com/AnimMouse/ffmpeg-stable-autobuild/releases) — builds are kept for two years.

## Features

- **Nonfree** — Fraunhofer FDK AAC (libfdk_aac) and DeckLink
- **Static** — all binaries are statically linked
- **Binaries** — ffmpeg.exe, ffprobe.exe, ffplay.exe
- **Architectures** — win64 (x86_64) and win32 (i686)

## How it works

Uses [ffmpeg-windows-build-helpers](https://github.com/rdp/ffmpeg-windows-build-helpers) to cross-compile FFmpeg for Windows on Linux runners. Version checks run against [endoflife.date](https://endoflife.date) for the latest FFmpeg stable release and the helper script's master commit. When either changes, a new build triggers automatically.

## Schedule

The workflow checks for updates every **Sunday at 11:07 UTC** and on **star events**. Builds only run when a new FFmpeg stable version or helper script commit is detected.

## Manual builds

Trigger a build via the **Actions** tab → **Build FFmpeg stable** → **Run workflow**:

| Input | Effect |
|-------|--------|
| `force_build` | Skip the version check — always build |
| `no_release` | Build and archive but skip creating a GitHub Release |

## Forking

Forks don't inherit Actions variables. Set `FWBH_REPOSITORY_OWNER` to point to a custom fork of `ffmpeg-windows-build-helpers` (defaults to `rdp`). Fresh forks start with no releases — the version check handles this gracefully.

## Related projects

- [ffmpeg-autobuild](https://github.com/AnimMouse/ffmpeg-autobuild) — nightly/snapshot builds from FFmpeg git master
- [My list of FFmpeg Binaries](https://www.animmouse.com/p/ffmpeg-binaries/) — community-built FFmpeg binaries

---

Powered by [ffmpeg-windows-build-helpers](https://github.com/rdp/ffmpeg-windows-build-helpers) · [GitHub Actions](https://github.com/features/actions) · [endoflife.date](https://endoflife.date)
