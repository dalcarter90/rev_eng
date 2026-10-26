# Prompt: Task 1, Windows hardware report (local Claude Code session)

Paste everything below the line into the local session. It's written for
the desktop; the Go 2 changes are at the end.

---

You're helping me test Project8Recomp v0.3.0 on real Windows hardware and
draft a "Windows report" for upstream. Project8Recomp
(github.com/theokyr/Project8Recomp, BSD-3-Clause) is a static recompilation
of Tony Hawk's Project 8 (Xbox 360). Upstream has only run v0.3.0 under
Proton, so a report from real Windows is the thing they want most. I play
the game; you handle setup, measurement, logs and the write-up.

This machine: desktop, NVIDIA RTX 5070 Ti, Windows 11.

## Rules
- No game content leaves this machine. That covers the ISO, `default.xex`,
  anything in the install's `game/` folder, assets, keys and generated
  code: none of it goes into anything you write, paste or upload. Hashes,
  sizes and log text are fine. Don't open files in `game/`.
- Before quoting logs, check them for anything copied from game files, and
  replace my user name and home path with `%USERPROFILE%`.
- Don't file the issue or post anything online. Draft it; I file it.
- Ask me before installing anything except PresentMon, before anything
  that needs admin, and before changing system settings (refresh rate,
  power plan, drivers).
- Work in `C:\p8test\`.

## Step 0: system facts
Collect: Windows edition, version and build (`Get-ComputerInfo`, plus
`winver` wording such as "24H2"); CPU; GPU and driver version
(`nvidia-smi`); RAM; resolution and refresh rate of the monitor the game
will open on, and whether G-SYNC/VRR is on; free space on the drive (the
install needs about 5 GB). Ask me for the ISO path and record only its
size.

## Step 1: get the release
Open github.com/theokyr/Project8Recomp/releases/tag/v0.3.0 and download the
Windows x86_64 archive. Tell me its exact name, size and SHA-256, and
whether the release publishes checksums to compare against. Unpack it to
`C:\p8test\thps-p8` (writable; not Program Files, not a OneDrive folder).
List the top level. Expect `Project8Recomp.exe`, `thps_p8_gui`, `thps_p8`,
`thps_p8_launch`, `thps_p8_identify`, `assets\`, `config\`, `logs\`.

## Step 2: measurement tools
Download the PresentMon console build (github.com/GameTechDev/PresentMon,
latest 2.x release) into `C:\p8test\tools`. Upstream reports p50 / p95
frame times and effective fps; PresentMon lets us report the same numbers.

Write `C:\p8test\tools\capture.ps1 -Seconds N -Label <name>` that:
- runs PresentMon on the game process (find its exact name while the game
  runs; probably `thps_p8.exe`) and writes a CSV to `C:\p8test\captures\`;
- samples total CPU % and the game process's CPU % once a second with
  `Get-Counter` (the process counter is per logical core, so divide by the
  core count to get a share of the whole machine);
- prints a summary: frames, seconds, effective fps (frames / seconds),
  mean / p50 / p95 / p99 / max frame time in ms (check the CSV header for
  the frame-time column name in this PresentMon version), frames over
  40 ms and over 50 ms, present mode, mean total CPU %, mean game CPU %.

Test it for 10 s on any window that's already running, before the game.

## Step 3: launcher (I drive, you take notes)
Start `Project8Recomp.exe`. SmartScreen should warn ("More info" then "Run
anyway"). Record: does the launcher open, and how fast; does the mouse
work; does a controller work (d-pad and stick move, A activates, B goes
back, bumpers cycle controls); anything visually wrong. Open the Display
screen and record the defaults (frame-rate cap should be on, Performance
on). Leave them.

## Step 4: setup
I point it at the ISO. Record: the identity check result and its exact
message, how long the copy takes (stopwatch), the final size of `game\`,
and whether `config\install.toml` was written. If the disc is refused as a
different release, stop and tell me: its `default.xex` hash and size
(never the file) is itself a contribution.

## Step 5: play, capped (defaults)
I boot to the main menu, Free Skate for 5+ minutes, then play one Career
level with an in-level cutscene. When I say so, run `capture.ps1` for
120 s in Free Skate and 60 s in the Career level. Also note:
- which GPU and present mode the game picked (from the logs);
- whether pacing looks even. If the monitor's refresh rate isn't a
  multiple of 30 (144 or 165 Hz, say), flag it: 30 fps can't pace evenly
  there unless VRR is on.

Timer check (needs admin, ask me first): while the game runs, run
`powercfg /energy /duration 15 /output C:\p8test\energy.html` and report
the "Platform Timer Resolution" entries for the game process (requested
period). v0.1.0 on Windows paced near 32 Hz because of the default 15.6 ms
timer; v0.2.0 switched to high-resolution timers, and this shows whether
that's in effect here.

## Step 6 (optional): uncapped
I turn the cap off in Display. Capture 60 s of Free Skate, and I'll watch
an in-level cutscene. Known upstream bug: in-level cutscenes run 3–5x too
fast uncapped. Confirm whether it happens here, and roughly how much
faster (I'll time the same cutscene capped and uncapped). Then I turn the
cap back on.

## Step 7: stopping
Once, stop the game with the launcher's Stop button (abrupt on Windows by
design, never tried on real hardware); once, quit from the game. Record
what happens, whether relaunching works, and whether the launcher offers
to free memory afterwards.

## Step 8: logs
Launcher home screen, Open logs folder. List the files with sizes. Pull out
GPU, driver, Vulkan, swapchain and present-mode lines, all warnings and
errors, and any crash info. Sanitize as above.

## Step 9: draft the report
Write `C:\p8test\report-desktop.md` following upstream's Windows report
template fields:
- **What happened** (one of): It worked / The launcher opened but something
  was wrong / The launcher would not start / Setup failed (disc check or
  copying) / It handed off to the game and the game failed / The game ran
  but performance or frame pacing was wrong.
- **Project8Recomp version:** v0.3.0
- **Windows version**
- **CPU and GPU** (plus GPU driver version)
- **Performance:** one line, e.g. "30 fps capped, 33.3 / 34.0 ms p50 /
  p95, 12% total CPU".
- **Details:** steps; launcher, setup and game observations; a table of
  each capture (scene, capped or not, effective fps, p50, p95, p99, max,
  frames over 40 ms, CPU); the timer check; the cutscene result; Stop and
  quit results. Compare with upstream's Proton numbers (capped 33.32 /
  34.25 ms p50 / p95; uncapped 27.01 / 30.99 ms, 36.30 effective fps) and
  say that their scene is a heavy late-game Funpark fixture, so it isn't
  like-for-like.
- **Output and logs:** relevant sanitized excerpts.

In your final message, paste the full draft, the raw capture summaries,
and anything that went wrong or that you skipped.

## Changes for the Legion Go 2
Use this instead of the "This machine" line and these overrides:
- This machine: Lenovo Legion Go 2, AMD Ryzen Z1 Extreme / Radeon 780M,
  Windows 11, built-in 1920x1200 144 Hz panel.
- GPU driver: AMD Software (Adrenalin) version, from
  `Get-CimInstance Win32_VideoController` or the AMD app.
- Record the power setup: on the charger or not, Legion Space performance
  mode and TDP. Run plugged in, in Performance mode.
- Step 5: capture the capped run first at 144 Hz (the default most users
  have), then ask me to switch the panel to 60 or 120 Hz and capture the
  same Free Skate stretch again. Note whether FreeSync/VRR is on.
- The game opens borderless fullscreen at desktop size: record whether its
  16:9 picture is stretched, letterboxed or pillarboxed on the 16:10 panel.
- Controller: use the built-in controls; note Legion Space's controller
  mode.
- Report file: `C:\p8test\report-go2.md`.
