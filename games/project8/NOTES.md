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
- Legion Go 2: Ryzen Z1 Extreme / Radeon 780M, Windows 11, 1920x1200 panel
  (set it to 60 or 120 Hz for 30 fps content).

Test plan per machine:
1. Download the v0.3.0 Windows archive and unpack it to a writable folder.
   Windows Defender's SmartScreen will warn because it's unsigned: More info,
   then Run anyway.
2. Launcher: does it open, can you navigate it (mouse and controller)?
3. Setup: point it at the ISO. Record whether the identity check passes and
   how long the 4.7 GB copy takes.
4. Game: boot to the menu, Free Skate for 5+ minutes, then one Career level
   with an in-level cutscene. Keep the default frame cap on.
5. Measure: fps (does it hold 30?), smoothness and stutter, total CPU % in
   Task Manager, GPU name in the logs.
6. Optional: turn the cap off and confirm the known cutscene speed-up
   happens on Windows too.
7. Collect the logs (launcher home screen: Open logs folder).
8. File the "Windows report" issue template, one per machine, ticking "no
   game files".

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
- [ ] ISO of own disc available
- [ ] Desktop Windows report filed
- [ ] Go 2 Windows report filed
- [ ] Linux build environment (WSL2) set up for Task 2
