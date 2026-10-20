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
  3. In progress: prototype the translator on a few hundred functions and
     check it against Ghidra's emulator.
  4. Next: the recompiled game boots in a stub runtime and logs its way to
     the first graphics call.
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

## First target: Legion Go 2 (Windows and Linux)
- Hardware: AMD Ryzen Z2 Extreme or Z1 Extreme (x86-64), RDNA 3.5 GPU,
  8.8" 1920x1200 (16:10) OLED **(verify for my unit)**.
- Build for `x86-64-v3` (hardware FMA, available on the Go 2 and any
  recent PC).
- Graphics: one Vulkan backend (SDL_GPU, or Vulkan directly) covers both
  Windows (AMD driver) and Linux (Mesa RADV). The runtime supplies its own
  NVN function table, using the 478 names in `nvn_procs.txt`, and
  translates the MFX shaders to SPIR-V. First check what MFX v0x35 holds:
  Maxwell binaries, or source/IR as well.
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
- What does `AppShaderPackage.mfx` (v0x35) contain: Maxwell shader
  binaries only, or source or IR as well?
- Does any code in `main` rely on unaligned access or on SWP/SWPB (old
  atomics)?
- How many threads does the game create, and which `nn::os`
  synchronization primitives does it use?
