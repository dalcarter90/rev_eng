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
  5. Next: graphics (scope below).
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

## Before first boot: checklist
- [ ] Unaligned and aliasing-safe accessors. Register and guest-memory
      accessors that cast pointers (`*(uint128_t *)...`) are undefined
      behavior under strict aliasing. Clang may also use aligned 16-byte
      SSE moves for `uint128_t`, which crash on guest addresses that are
      only 8-byte aligned (`vld1.64 {d0,d1}`). Use `memcpy`-based
      accessors, or typedefs with `aligned(1), may_alias`. Make `Ctx`
      16-byte aligned. Always build with `-fno-strict-aliasing`.
- [ ] Computed register offsets (`RX*`): assert in debug builds that they
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
