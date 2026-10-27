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

## Task 1 results: desktop
Local kit: `C:\rea-work\project8\kit` (`capture-session.ps1`,
`collect-report.ps1`), reports in `C:\rea-work\project8\reports\`.
- Kit bug, fixed locally: under Windows PowerShell 5.1, `$PSScriptRoot` is
  empty in `param()` defaults when run with `powershell -File`, so
  `Join-Path` failed. Set path defaults in the script body instead.

### Run 1 (first impressions, 2026-10-25 evening local)
Disc accepted, setup worked, game reached Free Skate (played as Nyjah
Huston). 2560x1440 desktop, launcher defaults (cap on, borderless
fullscreen at desktop size).
- **Picture doesn't fit the screen:** score HUD and most menus cut off;
  Options hard to use. Likely Windows-only, so worth the most to upstream.
  Proton/gamescope has no display scaling, so a DPI mismatch
  (window sized in scaled units, swapchain in pixels, or the reverse) is
  the first suspect; 1440p desktops usually run at 125% or 150% scale.
  To check: Windows scale %, then 100% scale, then windowed 720p / 1080p
  and explicit Fullscreen from the launcher's Display screen.
- **Heavy stutter, uneven frame times;** playable but hard. Unmeasured.
  To separate: first-run shader/pipeline compilation (better on a second
  run of the same area?) vs. pacing (monitor refresh rate not a multiple
  of 30, G-SYNC state) vs. a CPU stall.
- **Keyboard:** only ollies and spins, no tricks. Probably the keyboard
  map lacks the trick buttons, or it isn't documented. Use an Xbox
  controller for the measured runs, and find the keyboard map.
- **Capture `freeskate-capped`, 300 s, during that session** (PresentMon on
  the game process, `MsBetweenPresents`): 26,996 presents, 89.99 per
  second; mean and p50 11.11 ms, p95 11.22, p99 11.33; none over 40 ms.
  CPU: 31.3% average total (max 50%) on 12 logical cores; the game used
  2.52 cores on average (Deck under Linux: about 2.23).
  - The host presents at a rock-steady 90 Hz while the game is capped
    at 30, so each game frame is presented about 3 times. PresentMon counts
    presents, not new game frames, so this capture **can't see the
    stutter**: a late game frame shows as 4 presents, then 2, with even
    present timing. Why 90 Hz: unknown; check the monitor's refresh rate
    (a 180 Hz panel presenting every second refresh would give 90).
  - Game-frame timing needs the game's own counters (below).
- **Image quality:** shimmering, no AA, no visible resolution increase;
  shadows under the hair look pixelated. Probably the original 720p-class
  rendering and shadow maps, not a bug. A resolution-scale / AA option
  would be a feature request, not part of the report.

### Upstream tooling (from the v0.3.0 source, commit c8c21e2)
- Launcher Display "frame-rate cap" is the runtime's `--vsync` flag (guest
  vblank at the display rate when on, 1 ms when off). "Performance" adds
  17 runtime flags (`Settings::PerformanceFlags()` in
  `src/launcher/src/settings.cpp`). The launcher always passes
  `--mnk_mode=true` (keyboard input is disabled without it) and
  `--gpu_plugin=xenos`, and starts the game through `thps_p8_launch`.
- `--perf_log_csv=<path>` makes the game write `frames.csv`: one row per
  game frame with `frame_time_us`, draw counts and pipeline/texture cache
  misses. That's the right tool for the stutter (and pipeline misses test
  the shader-compilation theory). The launcher has no way to add extra
  flags, so: start the game normally, read its command line
  (`Get-CimInstance Win32_Process`), quit, then run `thps_p8_launch` with
  that command line plus the flag.
- F1 opens a debug panel (live frame time, fps, draws, cache misses, where
  the log and CSV go). It adds UI work every frame, so not during a
  measured run.
- F5 toggles native residency (one of the Performance paths).

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
