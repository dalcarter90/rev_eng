# Candidate games: Stage 0 findings

Results of the "does this already exist?" check (playbook Stage 0) for games
considered next. Re-check before starting; new projects appear often.

## Tony Hawk's Project 8 (Xbox 360 / PS3, 2006): already ported twice
Checked 2026-10-26.
- **Project8Recomp** (github.com/theokyr/Project8Recomp), BSD-3-Clause.
  v0.3.0 (Aug 2026). Static recompilation of the Xbox 360 version on the
  ReXGlue SDK.
  - Linux x86_64 verified (Steam Deck numbers in its README).
  - macOS arm64 verified on M1, but heavy scenes are slow.
  - Windows "experimental": tested only under Proton, because nobody on the
    project owns a Windows machine. They ask for reports from real Windows
    hardware.
- **ProjectRecomp** (github.com/gb92/ProjectRecomp), BSD-3-Clause, alpha.
  Also on ReXGlue (v0.10.0). Windows primary (Direct3D 12), Linux
  experimental (Vulkan). Reaches Career gameplay.
- Toolchain: ReXGlue SDK (BSD-3-Clause; Xbox 360 PowerPC to C++, with Xenos
  GPU shader translation), with credit to XenonRecomp/XenosRecomp
  (hedge-dev).

## Tony Hawk's Proving Ground (Xbox 360 / PS3, 2007): no port found
Checked 2026-10-26 with one web search. Same studio and engine generation as
Project 8, so the Project 8 ports' ReXGlue setup and engine knowledge would
carry over. Search more thoroughly (GitHub, the ReXGlue community) before
starting.

## Tony Hawk's Pro Skater 1–4 combined
- THUG Pro (a THUG2 PC mod) already combines levels from across the series.
- PARTYMOD patches exist for the THPS2/3/4 PC versions.
- Official remakes: THPS 1+2 (2020), THPS 3+4 (2025).
- PC versions exist, so this is the PC (incremental decompilation) path, not
  static recompilation.

## MHGU
See `games/mhgu/NOTES.md`. MHGURecomp (MIT) is fully playable.
