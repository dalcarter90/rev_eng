# Game porting process

A repeatable process for taking a PC game you own and porting it to
Windows and Linux first, then Android and iPad. It uses REA with Ghidra
for the native analysis.

The core idea: **you are not trying to read the whole binary.** You are
trying to:
1. separate the game's own logic from the platform it runs on (Windows,
   DirectX, input, audio, files), and
2. rebuild the logic as portable code on top of a portable platform layer
   (SDL3 and a portable renderer).

Every stage below has an exit check. Don't move on until it passes.

---

## Stage 0: Before decompiling anything (about an hour)

Most of the time saved comes from this stage.

1. **Check whether the work already exists.** Search for:
   - an official source release, or an official Linux, Android or iOS build;
   - an existing decompilation or reimplementation project
     (`<game> decomp`, `<game> reimplementation`, `<game> source port`);
   - an engine reimplementation that already runs the game, such as
     ScummVM, OpenMW, OpenRCT2, devilutionX or fheroes2.
2. **Check whether Linux already works.** Try the game under Proton or Wine.
   If it runs well, the Linux "port" is done for playing purposes, and the
   decompilation is only needed for mobile and ARM.
3. **Set the legal ground rules.** Write them into the game's `NOTES.md`:
   - Use only a copy you own. Never commit game assets (art, audio, data
     files) or the original executables to git. A port loads assets from
     the player's own copy at runtime.
   - Read the EULA. Decompiling for interoperability is tolerated in some
     places and forbidden by contract in others.
   - Publishing decompiled code is a legal risk you take on yourself.
     Keeping a port private carries far less risk than distributing it.
   - The App Store and Play Store will reject a port of a game you don't
     hold the rights to. Plan on TestFlight or sideloading for your own
     devices, unless the rights holder is involved.

**Exit check:** no existing port covers your goal, and `NOTES.md` records
the source of your copy, its version and its hashes.

---

## Stage 1: Triage. Identify the engine (about an hour)

The engine decides the whole approach. Look at the install folder first,
then ask REA: *"Use REA to inspect `<game folder>`. Identify the engine,
language, renderer, audio library and whether the main binary is 32- or
64-bit."*

| What you find in the folder | Engine | Port path | Difficulty |
|---|---|---|---|
| `*.pck`, `*.exe` with Godot strings | Godot | Recover the project with GDRE Tools, then export to every platform from Godot | Easy |
| `data.win` | GameMaker | Inspect with UndertaleModTool. Port through a GameMaker-compatible runner | Medium |
| `*.love`, or Lua files | LÖVE / Lua | It's a zip file. LÖVE runs on all targets | Easy |
| `Content/` folder plus a .NET exe referencing XNA, FNA or MonoGame | XNA/FNA/MonoGame | Decompile the C# (ILSpy or REA `inspect_managed_artifact`). Rebuild on FNA (desktop) or MonoGame (mobile) | Medium |
| `*_Data/Managed/Assembly-CSharp.dll` | Unity (Mono) | C# decompiles cleanly, but rebuilding a Unity project (AssetRipper) is heavy work | Hard |
| `GameAssembly.dll` plus `global-metadata.dat` | Unity (IL2CPP) | Recover names with Il2CppDumper or Cpp2IL first, then Ghidra | Very hard |
| `*.pak` plus `UE4`/`UE5` strings | Unreal | Treat as modding only. A full port is impractical | Impractical |
| A single large `.exe` with its own data formats | Custom C/C++ | **The full REA + Ghidra path below** | Hard, but the most reliable |
| A Java `.jar` | Java | Decompile, then port with LWJGL (desktop) or libGDX (mobile) | Medium |

Also note these down; they matter later:
- **32-bit or 64-bit.** Most older games are 32-bit x86, and every mobile
  target is 64-bit ARM. See "64-bit gotchas" below.
- **Renderer:** Direct3D 8/9/11, OpenGL or software rendering.
- **Audio:** DirectSound, XAudio2, FMOD, Wwise, Miles or OpenAL.
- **Middleware:** Bink video, PhysX, Steamworks, DRM.

**Exit check:** an engine row filled in, with the evidence (strings, DLL
names) written into `NOTES.md`.

The rest of this document covers the **custom C/C++** path, since it's
the one that needs Ghidra. The other engines reuse stages 5 to 7.

---

## Stage 2: Map the binary (a few days)

Goal: a labelled map of the binary. You don't need understood code yet.

Ask REA, keeping each answer in `NOTES.md`:
1. *"List imported DLLs and functions, grouped by subsystem (graphics,
   audio, input, files, windowing, timing, network)."* These imports are
   the **platform boundary**: everything you'll have to replace.
2. *"Find the entry point, WinMain and the main loop (the loop that calls
   PeekMessage/DispatchMessage, then update and render)."*
3. *"List strings that look like file names, extensions, config keys or
   error messages, and the functions that reference them."* Error strings
   are the fastest way to name functions.
4. *"Identify statically linked libraries"* such as zlib, libpng, Lua,
   the C runtime or STL. Mark them as **library code**. You will replace
   them with the real library, never decompile them.
5. Name what you find as you go, with REA's `set_address_name`,
   `set_comment` and `set_bookmark`. Unknown items go in with
   `record_unknown`.

Track every function in `function-map.csv` (template in
`games/_template/`). Each row records the address, name, subsystem, a
status (`unknown` / `named` / `decompiled` / `reimplemented` / `verified`),
and whether it's `game` / `platform` / `library` code.

**Exit check:** the main loop, the asset loader and every function that
calls a platform API are named, and every function is classified as
`game`, `platform` or `library`.

---

## Stage 3: Choose the decompilation strategy

| Strategy | What it is | Use it when |
|---|---|---|
| **Incremental replacement (recommended)** | Build a DLL that is injected into the original exe and replaces functions one at a time with your C/C++ code. The game keeps running the whole time, so each replaced function is tested in the real game straight away. Once everything is replaced, the original exe is no longer needed. | The default for custom C/C++ games. This is how projects like OpenRCT2 and re3 got started. |
| Matching decompilation | Write C so that compiling it reproduces the original bytes exactly, using the same compiler version. | You want byte-level proof of correctness, and you can find the original compiler. It's slow. |
| Clean reimplementation | Use the decompilation only as a reference and write a new engine that reads the original data files. | The original code is a mess, or you want a cleaner legal position (a "clean-room" approach). |
| **Static recompilation** | Translate every machine-code function into C automatically, compile it for the new platform, and write a runtime that replaces the original system (memory, threads, files, input, graphics). XenonRecomp (Xbox 360) and N64Recomp work this way. | Console games, and any game too large to decompile by hand. You get a playable port long before anything is understood, and you can then replace generated functions with readable code one at a time. |
| Emulation (Box64, Winlator, Wine) | Run the original x86 binary on ARM. | A stopgap only. It isn't a port, and it doesn't work on iOS. |

---

## Stage 4: Decompile and reimplement, one subsystem at a time (weeks to months)

Work from the outside in:
1. **File and asset loading.** Archive formats and decompressors. Write
   standalone extractor tools first. They prove you understand the formats
   and become the asset pipeline for the port.
2. **Platform layer.** Windowing, input, timing, audio and file I/O,
   behind your own interface (`platform.h`).
3. **Renderer.** Put a thin abstraction over whatever API the game used
   (see Stage 5).
4. **Game logic,** last and slowest: entities, AI, physics, UI, saves.

For each function:
1. Ask REA: *"Decompile `<function>`, name its locals and parameters,
   identify the structs it uses, and list its callers and callees."*
   Use `batch_decompile` for clusters of related functions.
2. Define the structs in a shared header, matching the original layout.
   Check the field offsets with `static_assert(offsetof(...) == 0x..)`.
3. Reimplement the function in C/C++, hook it in, and run the game.
4. Mark it in `function-map.csv` and commit. One subsystem per branch
   keeps the history readable.

**Testing.** This decides whether the project finishes.
- **Input recording and replay:** record a play session's inputs and RNG
  seed in the original. Replay them in your build and compare the game
  state frame by frame (positions, health, RNG state).
- **Save-file round trips:** load an original save, save it again, and
  diff the two files.
- **Screenshot diffs** of fixed scenes.
- REA `correlate_static_and_runtime`, plus Frida or `observe_native_calls`,
  to check what really executes when static analysis is ambiguous.

**Exit check per subsystem:** replay tests pass, and the subsystem's
functions are `verified` in the map.

---

## Stage 5: Windows and Linux build

- **Build system:** CMake, with one target per platform. Run CI builds for
  Windows (MSVC), Linux (GCC/Clang) and later Android and iOS from day
  one, so platform breakage shows up immediately.
- **Platform layer: SDL3.** It handles windows, input (including
  controllers and touch), audio, timing, files and app lifecycle on
  Windows, Linux, Android and iOS from one API.
- **Renderer:** pick a single target that covers all four platforms:
  - **SDL_GPU** (SDL3's API over Vulkan, Metal and D3D12), or **bgfx**,
    for a modern renderer;
  - **OpenGL ES 3.0** for a simpler 2D or older-3D game. It runs on
    desktop GL, Android and, via ANGLE, iOS.
  - Map the original Direct3D calls onto your abstraction. Don't
    translate D3D directly to Metal.
- **Linux specifics:** case-sensitive paths (normalize all asset lookups
  to lower case), `/` path separators, and saves stored under
  `$XDG_DATA_HOME` instead of the game folder.

**Exit check:** the game is playable start to finish on Windows and Linux
from the same source tree, with replay tests in CI.

---

## Stage 6: Android and iPad

The platform layer already runs there through SDL3. What's left:

- **CPU:** 64-bit ARM. Fix the 64-bit gotchas below before starting mobile.
- **Input:** design touch controls (an on-screen overlay or direct touch
  UI), support controllers through SDL, and handle a hardware keyboard
  on iPad.
- **Screens:** aspect ratios from 4:3 (iPad) to 20:9 (phones), notches
  and safe areas, and high DPI. Add a resolution-independent UI scale.
- **Lifecycle:** the OS can suspend or kill the app at any time. Save on
  `SDL_EVENT_WILL_ENTER_BACKGROUND`, and pause audio and rendering.
- **Assets:** the app ships without them. The player picks their own game
  folder: the Storage Access Framework on Android, the Files app or
  document picker on iOS. Copy the files into app storage on first run.
- **iOS rules:** no JIT or runtime code generation (this matters if the
  game embeds a scripting VM with a JIT), Metal only (through SDL_GPU or
  MoltenVK), and you need a Mac with Xcode to build. Install on your own
  devices with a developer account through Xcode, TestFlight or AltStore.
- **Android:** use the NDK with SDL3's Android project template. Test on a
  low-end device early, because mobile GPUs are far slower than desktop.

**Exit check:** playable on one Android device and one iPad, with saves
surviving the app being killed.

---

## 64-bit gotchas (32-bit x86 games)

These cause most of the bugs when a 32-bit game moves to 64-bit or ARM:
- **Pointer size.** Structs containing pointers change size and layout.
  Any struct read straight from a file or save must use fixed-width types
  (`int32_t`) and offsets, never raw pointers.
- **`long`** is 32-bit on Windows but 64-bit on Linux, macOS, iOS and
  Android. Use `<stdint.h>` types everywhere.
- **x87 floating point.** The original may compute at 80-bit precision, so
  physics and replays can drift slightly. Compare with a tolerance, or
  force consistent float behavior.
- **Undefined behavior the original relied on:** uninitialized memory,
  out-of-bounds reads, and MSVC-specific evaluation order. When a replay
  test diverges, suspect these first.
- **Alignment:** ARM is stricter about unaligned access in some
  situations. Use `memcpy` for unaligned reads from buffers.
- **Calling conventions** (`__stdcall`, `__thiscall`, `__fastcall`) only
  matter while hooking into the original exe. Note them in the function
  map.

---

## Console games (Switch, 3DS)

The stages are the same, with these differences:

- **Getting the files.** Dump the game and the keys needed to decrypt it
  from **your own console**. Never commit the dump or the keys, and never
  download either. The main code is in `exefs/` (`main`, plus `subsdk*`
  and `sdk`), and the assets are in `romfs/`.
- **Binary format.** Switch code is NSO, not PE. Use the Ghidra headless
  CLI with the SwitchLoader extension, which reads NSO directly. For large
  titles, prefer it to REA's Ghidra provider. Full steps and gotchas are in
  `docs/switch-re-pipeline.md`.
- **Free names at the platform boundary.** A Switch `main` calls the
  Nintendo SDK (`nn::fs`, `nn::hid`, `nn::os`, audio, NVN graphics) through
  dynamic imports with mangled C++ names. The whole platform boundary
  (Stage 2, step 1) is therefore already named. Demangle it and group it
  by subsystem.
- **CPU.** Switch games are ARM: AArch64, or 32-bit AArch32 for some
  titles (NPDM MMU flags bit 0 = 0). With static recompilation, guest
  memory keeps the original 32-bit layouts, so the pointer-size gotchas
  only hit hand-written runtime code, which must store guest pointers as
  `uint32_t`. AArch32 has its own translation traps (conditional
  execution, shift semantics, non-fused `VMLA`, NEON flush-to-zero). See
  `games/mhgu/NOTES.md`.
- **Graphics.** NVN is Nintendo's own API. Map it onto your renderer
  abstraction, the same way as Direct3D in Stage 5.
- **Reference runtime.** There's no original PC binary to hook into, so
  the incremental-replacement strategy (Stage 3) doesn't apply directly.
  Use an emulator as the reference instead. One with a GDB stub can be
  driven by REA's `start_gdb_session` for differential testing: run the
  same inputs in the emulator and in your build, then compare state.
- **Already playable through emulation?** Usually yes on Windows, Linux
  and Android. A native port is most valuable where emulation is weak:
  iOS/iPad (no JIT), low-end phones, and performance or mod support.
- **Legal.** Console makers enforce aggressively. Keep the work private.
  Never publish keys, dumps or assets, and treat publishing decompiled
  code as a serious risk.

---

## REA and Ghidra notes

- REA's Ghidra provider on Windows only reads **PE** files (`.exe` and
  `.dll`) and is read-only. To analyse a Linux (ELF) or Android (`.so`)
  build of a game, run REA inside WSL or on Linux.
- The first analysis of a large game binary can take a long time. Keep the
  Ghidra project between sessions rather than re-analysing.
- If a game ships with debug symbols (`.pdb`), or has a debug, demo or
  console build with more symbols, analyse that first. REA can then use
  `compare_functions` and `compare_application_versions` to carry names
  over to the release build.
- Export your findings regularly with `export_evidence_bundle`, and commit
  `NOTES.md` and `function-map.csv`. Never commit the binary itself.

---

## Per-game repo layout

```
games/<game-name>/
  NOTES.md            # source copy and hashes, engine, decisions, open questions
  function-map.csv    # every function: address, name, status, classification
  formats/            # one .md per file format you've decoded
  tools/              # extractors and converters (they read the player's copy)
  src/                # the port itself, once Stage 4 begins
```

Start a new game with:
`cp -r games/_template games/<game-name>`
