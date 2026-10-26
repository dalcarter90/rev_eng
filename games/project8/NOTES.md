# Tony Hawk's Project 8: contributing to Project8Recomp

Upstream: github.com/theokyr/Project8Recomp (BSD-3-Clause). It is a static
recompilation of the Xbox 360 version on the ReXGlue SDK. v0.3.0 (Aug 2026).
We contribute testing and fixes; we don't fork the port.

## Rules (from upstream CONTRIBUTING.md)
- **No game content, ever:** no ISO, `default.xex`, assets, keys, decrypted
  intermediates or generated code, in issues, PRs or this repo.
  `tools/check_no_game_content.sh` enforces it upstream.
- PRs to `src/`, `config/` and `patches/` are applied as patches through the
  development tree, with credit. `docs/`, `tools/` and `.github/` merge
  normally.
- Before a PR: build and test the launcher with ctest, then run
  `check_no_game_content.sh` and `check_patch_series.sh`.

## What we need
- An ISO of our own copy of the **supported release**: Tony Hawk's Project
  8, 2006 retail disc, title ID `0x415607DD`. Its `default.xex` has SHA-256
  `cfc732340e55defda400e25f03231aa9bb65fd9545b618212f69a4952384a5dd` and is
  8,237,056 bytes. Other regions are refused; their hash and size (never the
  file) is itself a useful contribution.
- About 5 GB free per install, and a Vulkan GPU.

## Task 1: Windows hardware report (their top request)
Upstream has only tested the complete v0.3.0 build under Proton. Report
from both machines:
- Desktop: RTX 5070 Ti, Windows 11.
- Legion Go 2: Ryzen Z1 Extreme / Radeon 780M, Windows 11, 1920x1200
  144 Hz panel. 33.3 ms isn't a multiple of a 144 Hz refresh, so 30 fps
  paces unevenly there (seen in the MHGU port). Test at 144 Hz (what a
  typical user has), then at 60 or 120 Hz.

Prompt for the local session: `prompts/task1-windows-report.md`.

Upstream baselines to compare against (v0.3.0, Proton 10 on a Steam Deck,
late-game Funpark fixture, ~2,600 draws per frame; our scenes differ):
- capped (default): 30 fps, 33.32 / 34.25 ms p50 / p95;
- uncapped: 27.01 / 30.99 ms p50 / p95, 36.30 effective fps;
- native Linux on the Deck, uncapped: 22.00 / 23.00 ms, 45.58 effective fps.

Measuring on Windows: PresentMon (Intel, open source) gives per-frame times,
so we can report the same p50 / p95 / effective fps; performance counters
(`Get-Counter`) give total and per-process CPU. `powercfg /energy` (admin)
lists each process's timer-resolution request, which shows directly whether
the v0.2.0 high-resolution timer fix is active on real Windows.

Also worth reporting (from upstream SETUP.md and KNOWN_ISSUES.md):
- Launcher defaults: the Display screen's frame-rate cap is on, and the
  Performance setting is on for a fresh install. Leave both as they are.
- The game opens borderless fullscreen at desktop size by default: check
  how 16:9 shows on the Go 2's 16:10 panel.
- The launcher's Stop button on Windows terminates the game outright. It
  has never been tried on real Windows hardware.
- Everything stays in the unpacked folder: `game/` (copied data), `saves/`,
  `config/settings.toml` and `install.toml`, `logs/`.

Test plan per machine:
1. Download the v0.3.0 Windows archive and unpack it to a writable folder.
   Windows Defender's SmartScreen will warn because it's unsigned: More info,
   then Run anyway.
2. Launcher: does it open, can you navigate it (mouse and controller)?
3. Setup: point it at the ISO. Record whether the identity check passes and
   how long the 4.7 GB copy takes.
4. Game: boot to the menu, Free Skate for 5+ minutes, then one Career level
   with an in-level cutscene. Keep the default frame cap on.
5. Measure: fps (does it hold 30?), frame times (p50 / p95), stutter, total
   CPU %, and the GPU name in the logs.
6. Optional: turn the cap off and confirm the known cutscene speed-up
   happens on Windows too.
7. Try the launcher's Stop button once, and a normal quit.
8. Collect the logs (launcher home screen: Open logs folder).
9. File the "Windows report" issue template, one per machine, ticking "no
   game files". The local session drafts it; I file it myself.

Relevant upstream history: v0.1.0 on Windows paced near 32 Hz because Windows'
default ~15.6 ms timer turned a 16.67 ms vblank sleep into 31.25 ms. v0.2.0
uses high-resolution timers. That's the same trap the MHGU port hit (fixed
there with `timeBeginPeriod(1)`), so watch frame times closely.

## Task 2 (later): fix "in-level cutscenes run 3–5x too fast uncapped"
- Upstream analysis: the only variable is the guest frame rate. Uncapped,
  the game runs at about 200 fps and the in-level cutscene update advances
  per frame. The front-end attract cinematic is fine, so it's different
  code. Next step: read the in-level cutscene update and find its time
  source.
- Building requires Linux (Windows builds are cross-compiled from Linux).
  On the desktop, use WSL2 (Ubuntu), or Bazzite on the Go 2. The game build
  needs the ReXGlue SDK plus `patches/rexglue-sdk/series`, then
  `rexglue codegen` against our own `default.xex` (generated code stays
  local).
- Approach: find what the cutscene update uses as its time step (a per-frame
  constant vs. a delta). Then compare with the front-end cinematic player,
  which upstream says is timebase-correct.

## Status
- [x] ISO of own disc available (region confirmed when the launcher checks it)
- [ ] Desktop Windows report filed
- [ ] Go 2 Windows report filed
- [ ] Linux build environment (WSL2) set up for Task 2
