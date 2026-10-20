# Monster Hunter Generations Ultimate (MHGU): notes

Process: see `docs/game-porting-process.md`, including "Console games".
Goal: native port. Windows and Linux first, then Android and iPad.

Items marked **(verify)** are expectations to confirm against your own
dump. They haven't been checked yet.

## Source copy
- Platform: Nintendo Switch (Capcom, 2018). Japan: Monster Hunter XX
  Nintendo Switch Ver. (2017). Monster Hunter XX for 3DS (2017, Japan
  only) is the same game on older hardware.
- Region / version / update:
- Dumped from my own console with:
- `exefs/main`, SHA-256:
- Other exefs modules (`subsdk*`, `sdk`), SHA-256:
- Never commit the dump, keys, `romfs/` assets or the NSOs.

## Stage 0: existing work and legal
- Existing decompilation or port projects searched:
- MHGU modding community tools for file formats (`.arc` archives etc.):
- Emulation already runs it on Windows, Linux and Android. The native
  port's main value is iPad, low-end Android and moddability.
- Legal: keep this private. Nintendo and Capcom both enforce.

## Stage 1: triage
- Engine: Capcom MT Framework (the Mobile branch, carried over from the
  3DS version) **(verify)**
- CPU: 32-bit ARM, per my own check. Record the exact evidence (NSO
  header, instruction set):
- Graphics: NVN **(verify)**
- Audio:
- Assets: MT Framework `.arc` archives in `romfs/` **(verify)**

## Stage 2: map. Starting points
- Platform boundary: demangle the `nn::` imports in `main` and group them
  by subsystem (fs, hid, os, audio, nvn).
- MT Framework naming: look for class-name strings from its runtime type
  system (MtDTI). The prefixes are `r` resources, `s` singletons/systems,
  `u` units, `c` other classes **(verify)**. They name thousands of classes
  for free.
- 3DS version as a second reference: the same game logic compiled
  differently. Use `compare_functions` to carry names across.
- Entry point:
- Main loop:
- Archive (`.arc`) loader:
- Statically linked libraries:

## Decisions
- Strategy: undecided. A full functional decompilation of MHGU is a
  multi-year project. Planned milestones, each useful by itself:
  1. Format tools: `.arc` extraction, then textures, models, quests, saves.
  2. A standalone SDL3 viewer that renders a model with its animations.
  3. Reimplement subsystems against the emulator as the reference
     (`start_gdb_session` for differential testing).
- Renderer target:

## Open questions
- Is `main` fully 32-bit, or are any `subsdk` modules 64-bit?
- Can REA load the NSO directly, or does it need converting to ELF first?
