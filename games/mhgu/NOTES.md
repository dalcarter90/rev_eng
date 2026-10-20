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
- CPU: unconfirmed, see open questions. Record the evidence (npdm flags,
  Ghidra language):
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
- Strategy: **static recompilation.** Translate all 43,754 functions in
  `main` to C, build for Windows, and provide a runtime for the Switch side
  (memory, threads, `nn::` services, NVN graphics, shaders).
- Progress:
  1. Done: extracted the game data, with the 1.4.0 update's data patch
     applied over the base files.
  2. Done: identified `subsdk0`.
  3. In progress: prototype the translator on a few hundred functions and
     check it against Ghidra's emulator. This settles register state, the
     memory model and the call table.
  4. Next: the recompiled game boots in a stub runtime and logs its way to
     the first graphics call.
- Reference projects: XenonRecomp (and Unleashed Recompiled, which is built
  on it) and N64Recomp.
- Renderer target:

## First target: Legion Go 2 (Windows and Linux)
- Hardware: AMD Ryzen Z2 Extreme or Z1 Extreme (x86-64, Zen 5 / Zen 4),
  RDNA 3.5 GPU, 8.8" 1920x1200 (16:10) OLED up to 144 Hz **(verify for
  my unit)**.
- CPU: x86-64, so the x86 translation hazards apply (division by zero,
  shift masking, FMA, NaN, float modes). Build for `x86-64-v3`, which
  covers the Go 2 and any recent PC and gives hardware FMA.
- Graphics: Vulkan is first-class on both Windows (AMD driver) and Linux
  (Mesa RADV), so one Vulkan backend covers both: SDL_GPU, or Vulkan
  directly. Translate the Maxwell shaders to SPIR-V.
- Frame rate: MHGU runs at 30 fps, and game logic is probably tied to the
  frame **(verify)**. Ship locked at 30 first. Higher frame rates need
  logic decoupled from rendering, which comes later.
- Aspect ratio: the Switch renders 16:9 and the Go 2 is 16:10. Start with
  letterboxing, and extend the camera and UI later.
- Input: use the SDL3 gamepad API. The Go 2 controllers appear as a
  standard gamepad (Legion Space on Windows; Steam Input or Handheld Daemon
  on Linux). Make the Nintendo A/B and X/Y swap a setting.
- Linux packaging: build inside the Steam Runtime (sniper) container and
  ship an AppImage or tarball that can be added to Steam as a non-Steam
  game. That covers SteamOS, Bazzite and CachyOS handheld installs.
- Build on a desktop, not the Go 2. Split the generated C across many
  files, use ccache or sccache, and build with `-O1` during development.
- Later idea: the 3DS version showed the map and item menus on a second
  screen. The dual-screen mount from the 3d_printing repo could bring that
  back as a second-window mode.

## iPad / iOS notes
- The generated C is the same for every platform. On iPad, Clang compiles
  it to ARM64 ahead of time, and the app is signed normally. No JIT is
  needed, which is why this approach works on iOS where emulators struggle.
- Raw Switch machine code can't run as-is, even though both are ARM64:
  - iOS won't execute unsigned code pages without JIT;
  - Apple reserves X18;
  - iOS uses TPIDRRO_EL0 as its own thread pointer;
  - the low 4 GB of address space is unavailable, while guest pointers are
    absolute Switch addresses.
  The C translation (context struct, `MEM()` macro) avoids all four.
- On ARM64 hosts most x86 hazards disappear (division by zero, FMA, NaN
  behavior and flags all match natively). Memory ordering is weak on both,
  so keep real fences for `DMB`.
- Page size: Apple uses 16 KB pages and the Switch 4 KB. Keep the runtime's
  guest mappings independent of the host page size.
- Entitlements to look into: increased memory limit and extended virtual
  addressing, needed for a multi-GB guest address space. Check what a
  free versus paid developer account allows.

## Open questions
- **32-bit or 64-bit? Settle this before the translator grows.** Check
  `exefs/main.npdm` (the 64-bit instruction flag and the address space
  type) and the language Ghidra loaded `main` as (`AARCH64` or `ARM:32`).
  The earlier "32-bit" note is unconfirmed.
- Can REA load the NSO directly, or does it need converting to ELF first?
