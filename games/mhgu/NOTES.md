# Monster Hunter Generations Ultimate (MHGU): notes

Process: `docs/game-porting-process.md` ("Console games").
Extraction steps: `docs/switch-re-pipeline.md`.
Goal: decompile `main` and recompile it as a native build. Legion Go 2
first (Windows and Linux), then Android and iPad.

Items marked **(verify)** haven't been confirmed in the binary yet.

## Source copy
- Nintendo Switch, title ID `0100770008DD8000`, SDK 4.4.1. Dumped from my
  own console.
- Target version: **1.4.0** (update title `0100770008DD8800`, v262144).
  The update replaces `main` entirely (14.29 MB). Work on `main_1.4.0`,
  not v0.
- `main_1.4.0` SHA-256:
- Japan: Monster Hunter XX Nintendo Switch Ver. (2017). Monster Hunter XX
  for 3DS (2017, Japan only) is the same game on older hardware.
- Never commit the NSP or NCAs, the ticket, keys, ExeFS or RomFS files.

## Local layout (Windows, `C:\rea-work\mhgu`)
- `nsp\`: v0 cnmt.xml, ticket, program NCA (12 GB). `exefs\`: v0 code.
- `upd140\`: 1.4.0 cnmt.xml, ticket, program NCA (202 MB) and `exefs\`.
- `romfs\nativeNX\`: RomFS with 1.4.0 applied (35,531 files, 12 GB,
  validated by magic bytes). File list in `romfs_paths.txt`.
- `ghidra\`: Ghidra project `mhgu` with `main` (v0, import only) and
  `main_1.4.0` (fully analyzed, the target).
- `survey\` and `scripts\Survey.java`: function and import survey.
- `keys\`: copies of `prod.keys` and `title.keys`. Never commit or share
  them.

## Stage 0: existing work and legal
- Existing decompilation or port projects searched:
- MHGU modding community tools for file formats (`.arc`, `.gmd`):
- Emulation already runs it on Windows, Linux and Android. The native
  port's value is performance, handheld fit (Go 2), iPad and mods.
- Legal: keep this private. Nintendo and Capcom both enforce.

## Stage 1: triage (confirmed)
- CPU: **32-bit ARM (AArch32)**. NPDM MMU flags bit 0 = 0, and the
  SwitchLoader selects `ARM:LE:32:v8`. 100% ARM-mode code, no Thumb.
- Load base: `0x60000000`.
- Engine: **Capcom MT Framework** (`Mt*` and `MtDti` strings).
- Graphics: NVN. Only `nvnBootstrapLoader` is imported. The other 478
  `nvn*` functions are looked up by name at runtime (`survey\nvn_procs.txt`).
- Shaders: a single `system\app_shader\AppShaderPackage.mfx` (1.3 MB, MFX
  v0x35).
- Audio: `.opus`. Movies: `.mp4`. Text: `.gmd`. Archives: `.arc`.

## Stage 2: map
- Survey of `main_1.4.0`: 43,754 functions, 13,489 computed jumps,
  40,464 computed calls (mostly vtables), 747 imports.
- Statically linked in `main`: NEX 4.4.0, Pia 5.2.15 and libcurl (online
  play). Stub them for an offline build; don't translate their internals
  beyond what's needed to boot.
- `subsdk0`: NintendoSDK movie 4.4.2. Replace it with FFmpeg rather than
  recompiling it.
- `sdk`: nnSdk 4.4.2, including NVN and zlib. Don't recompile it.
  Implement the APIs `main` imports in the runtime instead.
- MtDTI class names: the prefixes are `r` resources, `s` singletons/
  systems, `u` units, `c` other classes **(verify)**. Use them to name
  vtables and classes in bulk.
- 3DS version as a second reference, for carrying names across.
- Entry point:
- Main loop:
- Archive (`.arc`) loader:

## Decisions
- Tools: Ghidra headless CLI (`analyzeHeadless`, PyGhidra) with a
  persistent project and small post-scripts, rather than REA's Ghidra
  provider. This allows PDBs, a persistent database, compact output and
  batch jobs.
- Strategy: **static recompilation** of `main` to portable C, plus a
  per-platform runtime (memory, threads, `nn::` services, NVN, shaders).
  Gradual hand decompilation on top later.
- Progress:
  1. Done: extracted the game data, with the 1.4.0 update applied.
  2. Done: identified `subsdk0` (movie player).
  3. In progress: prototype the translator. Status (2026-10-20), on a
     2,000-function sample with random inputs:
     - **C matches Unicorn (QEMU ARM) on 1,979 functions, with 0
       mismatches.** Unicorn is now the primary reference and Ghidra's
       emulator the secondary one.
     - 21 functions are inconclusive. Unicorn can't map the top page
       (`0xFFFFF000`) and faults on misaligned exclusive loads, both only
       with garbage pointers. The other 19 are circular linked-list walks,
       mostly one container library around `0x6139xxxx`. They never end on
       random memory, even at 2 M steps with the "real" input profile
       (`RT_INPUTS=real`). They need inputs recorded from the running
       game.
     - The independent Unicorn reference exposed Sleigh errors that the
       Ghidra emulator shares:
       - `vcmp` sets C wrongly for unordered (NaN) compares. ARM sets
         C = !(a < b).
       - NaN priority and sign in `vmls`/`vnmls` differ from ARM's
         pseudocode.
       - 9 of the original 14 "C bugs" were Ghidra emulator errors.
       The translator now patches these per mnemonic.
     - NEON `vmax`/`vmin`/`vmaxnm`/`vminnm` lanes now flush to zero and use
       the default NaN, with ARM's signed-zero rules. Unhandled forms (Q
       registers, f16) raise `Unsupported`.
     - 186 tail branches in 183 functions land in non-entry code (shared
       epilogues). They're logged to `tails_nonentry.txt` and excluded for
       now. They need a translation strategy before step 4.
     - Tools: `unicorn_ref.py`, `compare.py`, `group_diffs.py`, and
       `harness.c` with `RT_DEBUG` traces and dumps.
     Full-program status (2026-10-20, later):
     - Export: 42,318 of 43,754 functions (everything except imports and
       Ghidra's external placeholders; 840 MB of p-code). An exporter bug
       had capped functions at 300 instructions, and internal thunks were
       mislabeled as imports. Both are fixed.
     - Translation: 42,547 functions, including 236 extra entry points for
       shared code. There are 7 failures: 3 were f64 `vsel`/`vrint` (now
       fixed) and 4 need 128-bit NEON. 90 shared-code branches in 21
       functions are still unresolved.
     - The whole translation compiles: 43 files plus the function table.
       Files with huge functions (up to 18,000 instructions) need fewer
       parallel compile jobs to avoid running out of memory.
     - Register jumps: switch tables become `goto`s. A jump to the LR the
       function was entered with is a return. Anything else is a tail call
       through the function table (`rt_jump`).
     - FPSCR: `main` never writes it (no `vmsr` in `.text`). The FZ/DN mode
       is a compile-time setting (`RT_FPSCR`), and both modes are tested.
       Which mode the Switch actually uses is still open.
     - Coverage on the 2,070-function sample: 1,450 functions at 90% or
       more, and 302 below 50% (not verified by a match alone). The sample
       includes 68 switch-table functions, 180 with indirect calls and 24
       over 300 instructions.
     - Harness: reserves 4 GB and commits pages when first touched, runs on
       a 1 GB stack thread, and stops a trial at a stack-depth guard
       (`DEPTH`, counted inconclusive).
     - Blocked until the SDK layer exists: functions that call `nn::`/libc
       imports.
     Two-mode verification (2026-10-21): both Unicorn runs were stopped
     when the PC ran low on memory, after about 59% of 16,560 trials. On
     the finished part (about 1,220 functions), each FPSCR mode gave:
     - 1,107 (FPSCR = 0) and 1,108 (FZ+DN) functions verified, meaning a
       match with at least 50% coverage;
     - 49 that match but have under 50% coverage;
     - **0 mismatches**;
     - 63 inconclusive, mostly indirect calls through null pointers in
       random memory ("target outside build").
     The memory growth came from keeping every lazily mapped Unicorn page.
     Fix: use a fresh Unicorn instance every few hundred functions, and run
     one mode at a time. Translation: 42,550 functions, 4 failing (128-bit
     NEON). `main` has no generic NEON float arithmetic, only `vmax`/`vmin`.
     SDK layer and linking (2026-10-21):
     - `nso_link.py` loads and links `main`, `subsdk0` and `sdk` the way
       `rtld` does, independently of Ghidra. Bases: main `0x60000000`,
       sdk `0x62800000`, subsdk0 `0x63200000`. Main's linked image is
       byte-identical to Ghidra's apart from the 2,051 import GOT slots.
     - Import split (main has 695 PLT imports):
       - 102 library imports (libc, libm, aeabi, C++ runtime,
         nn::util/crypto/codec) are **recompiled from `sdk`**: 185
         functions, bit-exact with the Switch. They reach only 4 runtime
         functions (mutex lock/unlock, condvar wait/broadcast).
       - The rest (fs, hid, audio, threads, sockets and so on) will be
         hand-written host implementations.
       - 17 imports resolve into `subsdk0` (movie player) and will be
         replaced with FFmpeg.
     - TLS reads (`mrc` TPIDRURO) become `Ctx.tls`, one per guest thread.
       The layout follows Horizon's ThreadLocalRegion (musl locale at
       +0x1e8, errno at +0x1ec).
     - Translator: 128-bit and 256-bit values, NEON integer, permute and
       crypto ops (`neon.h`), and register-space LOAD/STORE.
     - sdk library alone: 158 of 187 functions verified against Unicorn,
       0 mismatches.
     - Tail calls use clang `musttail`, including through the function
       table. Results on proto3 are identical on all 16,560 trials.
     - FPSCR: in `sdk` only fenv, `fma`, `lrint` and `nearbyint` write it,
       and none of those are in the build (the translator rejects
       `vmsr fpscr`). Thread start and `rtld` don't set it. Yuzu gives new
       threads FPSCR = 0 and copies the creator's value for created
       threads, so **`RT_FPSCR = 0` is the likely real mode** (not
       confirmed on hardware).
     - No stack switching in `main`: no fibers, no `setjmp`/`longjmp`, no
       C++ throw. The job system (`JobThread`) uses ordinary `nn::os`
       threads.
     - `ldrex`/`strex` currently always succeed (single-threaded tests).
       Real host atomics are needed before more than one guest thread runs.
     Design and commands: `C:\rea-work\mhgu\recomp\README.md`. The test
     harness is Windows-only, so verification stays on Windows; the game
     builds for both systems.
  4. **Done (2026-10-21): the recompiled game boots to its first graphics
     call.** It runs libc init, `nninitStartup`, main's 2,334 static
     constructors, `nnMain` and 4 guest threads, then stops at
     `nv::SetGraphicsAllocator`. That's 3,802 functions, and the run is
     deterministic. Details:
     - The boot runtime (`recomp/boot/`) replays sdk's `nn::init::Start`,
       which is what `rtld` calls. The sdk's own OS setup and constructors
       are skipped and logged, because host code replaces those services.
     - Host implementations so far: heap memory,
       `nn::mem::StandardAllocator`, ticks and sleep, read-only RomFS, and
       threads and sync (mutexes, condition variables, events, semaphores,
       message queues, timer events, TLS slots). Everything else goes to a
       logging stub. Calls are resolved at run time, so adding a host
       function needs no retranslation.
     - `ldrex`/`strex` (including `ldaex`/`stlex`) are a host
       compare-and-swap, and `dmb`/`dsb` are fences. Sleigh's `stlex`
       skipped the exclusive check entirely. Not yet verified against
       Unicorn.
     - Ghidra had missed about 26,000 functions reached only through
       pointers (vtables, callbacks), about a third of main. A data-pointer
       scan created them. It had also cut about 3,500 bodies short after
       calls wrongly marked as non-returning. Both are fixed.
     - The boot build is about 87,000 functions (all of main plus sdk). 104
       sdk crypto/NEON functions are untranslated and log if reached.
     - Open: the game asked for an 850 MB heap block that the 1.5 GB heap
       couldn't supply; it recovered with smaller blocks. 54 stub calls
       remain (oe, time, settings, swkbd, ngc init).
  5. **Done (2026-10-21): milestone A, the headless game loop.** NVN is a
     no-op, and the game runs anyway:
     - It reads `root.arc`, `resident.arc`, `resident_eng.arc` and
       `warning_eng.arc`, plus the shader package, system textures, a
       load-screen texture and the GUI font. English is selected.
     - 15 guest threads run on host threads (jobs, resource loader, sound,
       rendering, storage, NPad).
     - It runs about 30 fps (1,810 frames in 81 s) with about 16 draws per
       frame. It sets up 6,311 shader programs and about 19,000 buffers.
     - It sits on what's probably the warning screen (6,804 functions),
       waiting for input or a timer.
     Verification: proto5 (new code plus 58 functions with atomics) gave
     973 verified and 0 mismatches. All 26 atomic functions that ran
     match. proto4's partial run: 660 verified, 0 mismatches.
     Bugs fixed on the way:
     - `Tick`/`TimeSpan` are returned through memory (sret, the result
       pointer in r0). Returning them in registers broke the frame limiter.
     - `__aeabi_uidiv` jumps into an unrolled loop Ghidra never
       disassembled. Fixed with `AddJumpTargets.java`.
     - Bodies cut short after calls: the repair now repeats until nothing
       new turns up.
     - `GetAudioRendererWorkBufferSize` makes an IPC call, so it now has a
       host implementation.
     The accessors are aliasing- and alignment-safe (`may_alias`,
     `aligned(1)`, `_Alignas(16)`, `-fno-strict-aliasing`), and `RX*`
     offsets are checked. Of the old unresolved tail branches, 2 remain,
     both data. Guest layout: heap at `0x80000000` (2 GiB, yuzu's layout
     for address-space type 2), arena `0x42000000`, NVN fake procs at
     `0x50000000`.
  6. **Done (2026-10-21): milestone B, NVN host state.** Details:
     - Scripted input (`RT_INPUT="400:A,..."`) gets past the warning
       screens to the title. The game loads `title.arc`/`title_eng.arc`
       and creates its save data (system and system_backup, about 5 MB) in
       the host save folder.
     - NVN objects are tracked by guest address (about 148,000 live at the
       title). Command lists are recorded and submitted: about 200
       commands and 10 draws per frame at the title. `RT_NVN_LOG=<frame>`
       logs one frame.
     - Graphics allocator and out-of-memory callbacks work through
       host-to-guest calls.
     - **Shader dump (`RT_SHADERS=1`):** 6,311 programs, each with a vertex
       and a fragment stage, all created at load from
       `AppShaderPackage.mfx`. 12,606 unique stages, 29.9 MB of code (256 B
       to 5 KB each, median 2.3 KB). Format:
       - the control section has magic `0x98761234` and is 0x848 bytes,
         with the code size at +0x704;
       - the code starts with a 0x30-byte header (magic `0x12345678`),
         then the 0x50-byte Maxwell shader program header (SPH), then
         instructions and constants.
     - Fixed on the way: 149 callback functions (taken by PC-relative
       `adr`/`ldr+add pc`) had never been created. Frame rate: Windows
       rounds 1 ms sleeps up to 15.6 ms, so `timeBeginPeriod(1)` was needed
       (otherwise 22 fps). With it, the game holds exactly 30 fps and the
       main thread is idle 93% of the time.
     - **Blocker:** the title scene plays `movie\movie06.mp4` and accepts
       input only once the movie reports "playing". The movie library
       (`subsdk0`) isn't run, so a host movie layer is needed.
     - Unknown values: `nvnDeviceGetInteger` (17 queries) and texture
       storage size/alignment. Both are needed for C and E.
  7. **Done (2026-10-22): movie stand-in, NVN queries, milestone C.**
     - Movie stand-in (`hle_movie.c`): host Extractor/Decoder/MediaData/
       Buffer for subsdk0's MediaCodec-style API. The extractor reads the
       `.mp4` header (movie06: H.264/AAC, 960x540, 29.97 fps, 5,280
       frames, 176 s). The decoder emits black NV12 frames with real
       timestamps. Video track only, so movies are silent for now.
       Movies end on their own, and A skips them.
     - Past the title into the menus and character creation. The game
       loads player-model archives and title music, and writes its save.
       **Next blocker: hunter-name entry** (software keyboard stub, and
       `swkbd_eu.arc` is not found).
     - NVN queries: the runtime asks the stripped driver in `sdk` for the
       real address of all 477 entry points (`nvn_closure.py`):
       - 316 are pure; 196 of them now run as recompiled sdk code
         (builders, getters/setters, `nvnDeviceGetInteger`, texture
         handles, storage size/alignment);
       - 140 reach the OS and stay hand-written;
       - 21 make indirect calls.
       Real values include API 53.105 and uniform alignment 256.
       `NvRmGpuDeviceChooseKind` is hand-written. GetInteger 24 and 56 read
       device fields set over IPC; they stay 0 for now.
     - Texture-size bug: the `.tex` data is padded to the hardware's sizes,
       and a generic page kind undersized some textures, which overran the
       game heap. Current fix: every texture is rounded up to 128 KB.
     - Milestone C (`present_vk.c`): SDL3 window plus a Vulkan swapchain,
       tested on an RTX 5070 Ti. Each frame shows its clear color (black
       on these screens; a forced color proved the path works).
       `RT_HEADLESS=1` runs without a window.
     - D assessment: Ryujinx's shader translator fits the dumped stages
       (strip the 0x30 NVN header). Inputs (texture types, attribute
       types, binding slots) come from NVN state and the 0x848-byte
       control section. Plan: record per-program state during play, then
       translate offline. License: MIT per the Ryubing project page; the
       LICENSE file is not yet verified.
  8. **2026-10-22 round (stopped for review before shader translation):**
     - Texture kinds: plain color textures use the generic page kind,
       sized exactly. Depth/stencil and compressible ones are padded. All
       48,238 `.tex` files match the generic sizes. Padding everything
       would have cost +31% overall (player equipment: 476 MB becomes
       2.6 GB).
     - The earlier heap corruption was really `nvnDeviceGetInteger(56)`
       (linear texture stride alignment) returning 0. It's now 32
       (hand-written). Padding had only hidden the bug.
     - Name entry: host inline and full-screen keyboards type `RT_NAME`.
       The rest of character creation can't be navigated blind.
     - Shader control section: code length +0x6f8, register count +0x70c,
       stage +0x714, texture binding count +0x7f9, unknown permutation
       +0x804. Uniform binding b maps to c[b+3]. Texture binding b's handle
       is at 0x20+8b in the stage's driver constant buffer, probably c[2]
       as Ryujinx assumes for NVN (verify).
     - `RT_PROGSTATE=1` records per-program draw state: 97 combinations up
       to character creation.
     - First rendering target: the warning screen, 10 programs (20
       stages), all 2D quads with one attribute and one texture.
     - Waiting on: a local Ryujinx source copy for translation.
     - Translator choice: Ryujinx (C#, MIT) or Eden (a yuzu fork; C++,
       GPLv3). Eden's recompiler fits the C/clang toolchain and could also
       run inside the runtime as a fallback for unrecorded shader states,
       including on Android. Linking GPL code into the runtime makes the
       runtime GPL, which only matters if it's ever distributed. Both
       work for the Ryujinx-or-Eden save trick.
  9. **Done (2026-10-22): milestone D for the warning screen, and a real
     save loads.**
     - `shader_tool/nvn_shaderc` runs Eden's shader recompiler offline. It
       is GPL-3.0-or-later; the runtime doesn't link it. The vendored Eden
       subset is pinned by content hash (the download had no commit), and
       the dependencies (fmt, sirit, SPIRV-Headers, boost) were checked
       against Eden's SHA-512 hashes.
     - Bindings confirmed from the real sdk code: texture handles are in
       c[2] (the driver sets method 0x982 to 2, Eden's "proprietary driver"
       case), and uniform binding b maps to c[b+3]. c[1] holds the
       compiler's constants (control +0x6fc size, +0x700 offset); Eden
       folds them into the shader.
     - Warning screen: 13 recorded states become 10 programs (9 textured 2D
       quads and one FXAA pass). All 20 SPIR-V modules pass `spirv-val`
       with `uniformBufferStandardLayout` on. GLSL dumps are kept for
       reading.
     - **A save made in Eden loads.** The game loads the saved character
       (hair 010, armor 663), skips character creation and enters village
       v01. It runs 6,000 frames headless with no crash.
     - Fixed on the way: 382 switch jump tables that Ghidra missed or
       garbled (357 main, 25 sdk) were decoded from the binary and checked
       against their bounds checks, then 104 functions were re-exported.
       Also fixed: one missed sdk `vsscanf` callback, and duplicate jump
       targets in the translator.
     - Still to record for E: NVN depth mode and window origin (y-flip).
  10. E started (2026-10-22):
     - **All 6,311 programs translate**, with default state (16 float
       attributes, 2D textures). They reduce to 6,303 distinct vertex and
       fragment pairs, with no geometry or tessellation stages. All 12,606
       SPIR-V modules pass `spirv-val`.
     - Command set in use: state binds, clears, indexed and non-indexed
       draws, barriers. No copy or blit commands.
     - Depth mode and window origin are set once each
       (`nvnDeviceSetDepthMode`, `nvnDeviceSetWindowOriginMode`) and are
       already stored on the device object.
     - Next: texture format list (ASTC check), the renderer (buffer copies
       at submit, MIT deswizzle, texture cache, pipelines from the offline
       SPIR-V), then a warning-screen screenshot.
- Reference projects: XenonRecomp (and Unleashed Recompiled, which is built
  on it) and N64Recomp.

## Translator design for AArch32
- **Context struct:** R0–R12, SP, LR; the N, Z, C, V and Q flags (keep
  them as separate bytes); the GE flags if SIMD-in-GPR instructions occur;
  the VFP/NEON register bank as a union, because S0–S31, D0–D31 and Q0–Q15
  overlap (S2n and S2n+1 are the halves of Dn, and Q n is D2n:D2n+1);
  FPSCR; and the TLS register TPIDRURO (read with `MRC p15, 0, Rx, c13,
  c0, 3`). One context per guest thread.
- **Memory:** guest pointers are 32-bit, so reserve one 4 GB host region
  and define `MEM(addr) = base + (uint32_t)addr`. Overflow wraps for free,
  and this works the same on x86-64 and ARM64 hosts, including iOS.
  Recompiled code keeps the game's original 32-bit struct layouts in guest
  memory, so the pointer-size problem only affects hand-written runtime
  code. That code must hold guest pointers as `uint32_t` and never as
  host pointers.
- **Call table:** a flat array indexed by `(addr - 0x60000000) / 4`. ARM
  mode only, so every function start is 4-byte aligned. 14.29 MB of
  code is about 3.7 M slots, or about 30 MB of host pointers. Unknown
  targets log and trap.
- **Semantics to get right**, each with a unit test against Ghidra's
  emulator:
  - Conditional execution: almost every ARM instruction can carry a
    condition, so emit `if (cond)` and track flags carefully. Only compute
    flags that are later read.
  - Barrel shifter: the shifted operand produces its own carry-out, which
    sets C on flag-setting logical instructions.
  - **Register-specified shifts use the bottom byte of the register, not
    the amount mod 32.** LSL/LSR by 32 or more gives 0. ASR by 32 or more
    gives all sign bits. ROR uses the amount mod 32. (In C, a shift of 32
    or more is undefined behavior, so guard it.)
  - Reading PC gives the current instruction's address + 8.
  - Returns and branches come in many forms: `BX LR`, `POP {..., PC}`,
    `LDM ... {PC}`, `LDR PC, [...]`, `MOV PC, Rx`. They need classifying
    as return, tail call or jump table.
  - SDIV/UDIV return 0 on division by zero, whereas x86 traps.
  - **`VMLA`/`VMLS` are not fused** (multiply, round, then add) and
    `VFMA`/`VFMS` are. Compile with `-ffp-contract=off` and use `fma()`
    only for the fused forms.
  - **NEON (Advanced SIMD) float ops always flush denormals to zero and
    use the default NaN**, whatever FPSCR says. VFP scalar ops follow
    FPSCR. Emulate both.
  - Saturating instructions (`QADD` and similar) set the Q flag.
  - `LDREX`/`STREX` (and `LDREXD`/`STREXD`) become compare-and-swap
    loops. `DMB` becomes a real fence (needed on ARM hosts).
  - `SVC`: there shouldn't be any in `main`, because system calls live in
    `sdk`. Any that turn up go to the runtime.

## Next: getting to the title screen, in milestones
A. **Headless game loop.** Graphics is a no-op, and the goal is for the main
   loop to run frame after frame without a window. This finds every
   non-graphics blocker first:
   - display: `nn::vi` display and layer calls, and a vsync event the
     runtime signals at 30 or 60 Hz, or the loop stalls;
   - audio: an `nn::audio` renderer stub that keeps its timing events
     ticking;
   - input: `nn::hid` reporting one connected controller with nothing
     pressed, and controller-applet calls returning success;
   - settings: `nn::oe`/settings reporting English, plus time;
   - saves: `nn::fs` save data mounted on a writable host folder.
   Success means `.arc` files are read and frames keep counting.
B. **NVN as logging stubs.** The game gets its 478 NVN functions from
   `nvnBootstrapLoader` / `GetProcAddress` and calls them through pointers.
   Hand out fake guest addresses (one reserved range) that the function
   table maps to host functions. Graphics callbacks such as
   `nv::SetGraphicsAllocator` need host-to-guest calls, made the same way
   threads start guest code. Log the call sequence of one frame.
C. **Window and clear color.** SDL3 window plus a Vulkan swapchain. Map
   NVN queue, command buffer and present calls far enough to show the
   game's clear color each frame.
D. **Shaders.** Find `system\shader\ShaderPackage` (probably in an
   `.arc`), identify the binary format, and translate the shaders to
   SPIR-V ahead of time.
E. **Draws, textures, render targets,** until the title screen appears.

Saves: the game code is identical to the Switch's, so a save dumped from
your own Switch should load unchanged once save data is mounted.

## Before first boot: checklist
- [x] Unaligned and aliasing-safe accessors. Register and guest-memory
      accessors that cast pointers (`*(uint128_t *)...`) are undefined
      behavior under strict aliasing. Clang may also use aligned 16-byte
      SSE moves for `uint128_t`, which crash on guest addresses that are
      only 8-byte aligned (`vld1.64 {d0,d1}`). Use `memcpy`-based
      accessors, or typedefs with `aligned(1), may_alias`. Make `Ctx`
      16-byte aligned. Always build with `-fno-strict-aliasing`.
- [x] Computed register offsets (`RX*`): assert in debug builds that they
      stay inside the VFP/NEON bank (`0x300`–`0x3ff` including the access
      size), instead of masking with `& 1023`, which silently wraps.
- [x] Real `ldrex`/`strex` (`ldrexd`/`strexd`, `ldaex`/`stlex`) before a
      second guest thread runs: a per-thread reservation (address and
      value) in `Ctx`, and `strex` as a host compare-and-swap.
- [x] Startup path: run each module's init (`DT_INIT`/`.init_array`) with
      dependencies first (sdk, then subsdk0, then main). Then follow the
      real entry (sdk's startup, `nninitStartup`, `nnMain`) instead of
      calling `nnMain` directly. MT Framework probably registers its
      class/DTI tables in static constructors.
- [x] (Probably covered by the function-creation passes; recheck the list.) The 90 unresolved tail targets (21 functions): have Ghidra
      disassemble them and create functions where they're real code.
      Re-export.
- [ ] OS-level imports (about 250): hand-written shims at the `nn::` API
      level, starting as logging stubs. Emulators implement the kernel and
      IPC services *below* the SDK, so their service layers don't plug in
      here. Use them as behavior references only, and check licenses
      (yuzu GPLv3, Ryujinx MIT).

## First target: Legion Go 2 (Windows and Linux)
- Hardware: AMD Ryzen Z2 Extreme or Z1 Extreme (x86-64), RDNA 3.5 GPU,
  8.8" 1920x1200 (16:10) OLED **(verify for my unit)**.
- Build for `x86-64-v3` (hardware FMA, available on the Go 2 and any
  recent PC).
- Graphics: one Vulkan backend (SDL_GPU, or Vulkan directly) covers both
  Windows (AMD driver) and Linux (Mesa RADV). The runtime supplies its own
  NVN function table, using the 478 names in `nvn_procs.txt`.
- Shaders:
  - `system\app_shader\AppShaderPackage.mfx` (1.3 MB, MFX v0x35) is
    **metadata only**. It has 2,862 named entries:
    - 1,773 shader functions, 249 passes and 68 techniques;
    - 166 constant buffers and 161 texture slots;
    - blend, depth and raster states, plus a string table.
    Names like `$Globals` suggest the source is HLSL.
  - There's no Maxwell code or NVN shader header in the MFX or in `main`.
  - The executable also loads `system\shader\ShaderPackage`, which isn't
    a loose file, so it's probably inside an `.arc`. It goes with the
    `mpShaderArchive`, `nNX::ShaderBinary` and `rShader2` resources, and
    the compiled shaders are most likely there. Plan: translate the shaders to SPIR-V once, ahead of time,
  so there's no runtime stutter. Some Maxwell shaders may need variants
  per pipeline state (texture types, vertex formats), which is why
  emulators translate at runtime. Emulator shader recompilers are a
  possible reference or offline tool, but check their licenses (often
  GPL).
- Linux runtime differences: `mmap` instead of `VirtualAlloc` for the 4 GB
  guest region, plus paths and threads. A few hundred lines. The Windows
  build under Proton on Bazzite is a fallback only.
- Frame rate: MHGU runs at 30 fps, and game logic is probably tied to the
  frame **(verify)**. Ship locked at 30 first.
- Aspect ratio: the Switch renders 16:9 and the Go 2 is 16:10. Letterbox
  first.
- Input: SDL3 gamepad API, with the Nintendo A/B and X/Y swap as a setting.
- Movies: FFmpeg for `.mp4`. Audio: libopus.
- Linux packaging: build in the Steam Runtime (sniper) container. Ship an
  AppImage or tarball that can be added to Steam as a non-Steam game.
- Build on a desktop. Split the generated C across many files, use ccache
  or sccache, and build with `-O1` during development.
- Later idea: the 3DS version showed the map and item menus on a second
  screen. The dual-screen mount from the 3d_printing repo could bring that
  back as a second-window mode.

## Later: resolution and frame rate
Do these after the game runs as-is, but keep them possible from the start:
- Build the renderer so that every render target is created through one
  function with a resolution scale factor. Don't assume 16:9 anywhere.
- Keep frame pacing (present, vsync, frame timer) inside the runtime.
- During step 4, log NVN render-target sizes, and find how the game reads
  time and vsync (`nn::os` ticks, `nn::vi` vsync events). Those are the
  hooks for both features.

Resolution (moderate effort): scale the render targets the game asks for,
the way emulators do. Watch for:
- post effects that assume pixel sizes (blur, bloom, depth of field,
  ambient occlusion);
- UI drawn in fixed pixels;
- shadow maps, which should be scaled separately;
- texel offsets passed in constant buffers (the MFX metadata lists them).

Frame rate (harder): the game runs at 30 fps, and its logic (animation,
physics, timers) probably counts frames **(verify)**. In rising order of
effort:
1. Driver frame generation (AMD Fluid Motion Frames on the Go 2). No game
   changes, but it adds latency.
2. Find the game's timestep or frame-rate setting. Check existing 60 fps
   patches from the emulator community, if any, since they show which
   values to change. Then fix whatever runs at double speed.
3. Run logic at 30 Hz and interpolate rendering between logic ticks. This
   is the smoothest result and the most engine work.

## Later: Android and iPad
- The generated C and the 4 GB `MEM()` model carry over unchanged to
  ARM64.
- **Apple chips (M-series, A11 and later) cannot run AArch32 code at all**,
  so translation is required, not optional. The same is true of many
  recent Android chips.
- iOS: no JIT is needed, because the recompiled code is signed into the
  app. The 4 GB guest region still needs base + offset, since iOS reserves
  the low 4 GB. Look into the increased-memory-limit entitlement.
- Page size: Apple uses 16 KB pages and the Switch 4 KB. Keep guest
  mappings independent of the host page size.

## Open questions
- Where is `system\shader\ShaderPackage`, and are its shaders raw
  Maxwell, NVN shader binaries, or something else?
- FPSCR: `RT_FPSCR = 0` per yuzu's thread model. Confirm on hardware if
  that's ever cheap to do. Both modes pass the tests.
- Does any code in `main` rely on unaligned access or on SWP/SWPB (old
  atomics)?
- How many threads does the game create, and which `nn::os`
  synchronization primitives does it use?
