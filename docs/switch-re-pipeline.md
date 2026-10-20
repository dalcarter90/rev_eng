# Switch extraction and analysis pipeline (Windows)

Steps that worked for MHGU (October 2026). Use only dumps and keys from
your own console. Never commit or share keys, title keys, tickets, NSPs,
NCAs or extracted files.

## Tools
- Ghidra 12.1.x with JDK 21, run headless (`support\analyzeHeadless.bat`,
  `support\pyghidraRun.bat`).
- Adubbz SwitchLoader extension for Ghidra. The latest release targets
  Ghidra 12.1.2. To use it with 12.1.4, install it under
  `%APPDATA%\ghidra\ghidra_12.1.4_PUBLIC\Extensions\SwitchLoader` and patch
  the version in `extension.properties` to 12.1.4. Headless mode picks it
  up.
- `hactool` 1.4.0: for the base game's ExeFS only (see gotchas).
- `nstool` 1.9.2: for game data with an update applied.
- `pfs0.py`: a small script that lists and extracts NSP (PFS0) entries.
  It took about 17 s for a 12 GB NSP.
- Avoid unofficial mirror binaries. The official LibHac/hactoolnet
  repository is gone from GitHub.

## Steps
1. **Unpack the NSP.** `python -I pfs0.py <nsp>` lists the entries, and
   `pfs0.py <nsp> <name> <outdir>` extracts one. (`nstool -x` for single
   files failed on Windows paths.)
2. **Find the program NCA** in the `.cnmt.xml`. The title key is ticket
   bytes `0x180..0x190`, and the rights ID is ticket bytes `0x2A0..0x2B0`.
   Don't print title keys to logs or chat.
3. **Base ExeFS:**
   `hactool -k <prod.keys> --titlekey=<hex> --exefsdir=exefs <program.nca>`
4. **Game data with an update applied:**
   `nstool -k <prod.keys> --basenca <base program.nca> -x /1 <outdir> <update program.nca>`.
   nstool reads title keys only from `%USERPROFILE%\.switch\title.keys`,
   in the format `<rights id> = <title key>`.
5. **Ghidra:** run `analyzeHeadless.bat` from PowerShell. Import
   `exefs\main` once into a persistent project. After that, use
   `-process <program> -noanalysis` with small Java or PyGhidra
   post-scripts that write only the output you need.

## Gotchas
- **hactool 1.4.0 silently corrupts update-patched (BKTR) RomFS output.**
  It writes garbage bytes, then fails partway with "Failed to read file!".
  Never use it for base + update data; use nstool. Validate every
  extraction by magic bytes (for MT Framework: `ARC\0`, `GMD\0`, `GUI\0`,
  `TEX\0`).
- Git Bash rewrites arguments that start with `/` into Windows paths. Set
  `MSYS_NO_PATHCONV=1` for tools that take in-container paths like `/1`.
- Quoting the `.bat` path breaks under bash and cmd. Run Ghidra headless
  from PowerShell.
- The NPDM's MMU flags bit 0 is the 64-bit flag. When it's 0, the title is
  32-bit and SwitchLoader selects `ARM:LE:32:v8`.
- Only delete a stale `<project>.lock*` when no Ghidra Java process is
  running.
- Prefer the headless CLI to REA's Ghidra provider for large titles. It
  supports caller-supplied PDBs, keeps a persistent database, gives
  compact script-controlled output and runs batch jobs.
