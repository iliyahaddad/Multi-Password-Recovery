# Multi Password Recovery - Build Guide

## Prerequisites

| Tool | Path | Version |
|------|------|---------|
| MASM32 | `C:\masm32` | MASM 5.12.8078+ |
| Polink (Pelle) | `C:\masm32\bin\polink.exe` | Included with MASM32 |
| RC (Resource Compiler) | `C:\masm32\bin\rc.exe` | Included with MASM32 |

**MASM32 must be installed at `C:\masm32`** — the build script uses this fixed path.

## One-Command Build

```cmd
cd C:\path\to\Multi Password Recovery
build.bat
```

**Output:** `Multi Password Recovery.exe` (in project root)

## What build.bat Does

```batch
1. cd src
2. Assemble all 14 modules:
   ml.exe /c /coff MultiPasswordRecovery.asm core.asm browsers.asm
       winmods.asm apps.asm json.asm pstorage.asm opera.asm
       ftpclients.asm imclients.asm mailclients.asm
       credfiles.asm lsasecrets.asm sysvault.asm
3. Compile manifest resource:
   rc.exe /fo mpr.res MultiPasswordRecovery.rc
4. Link with Pelle Polink:
   polink.exe /SUBSYSTEM:WINDOWS
       MultiPasswordRecovery.obj core.obj browsers.obj ...
       mpr.res
       lib32/pf_kernel32.lib lib32/pf_user32.lib ...
       lib32/pf_ws2_32.lib
       C:\masm32\lib\masm32.lib
5. Output: ..\Multi Password Recovery.exe
```

## Required Files (Must Exist)

### Source (`src/`)
```
MultiPasswordRecovery.asm   core.asm        browsers.asm      winmods.asm
apps.asm                    json.asm        pstorage.asm      opera.asm
ftpclients.asm              imclients.asm   mailclients.asm   credfiles.asm
lsasecrets.asm              sysvault.asm    pv.inc
MultiPasswordRecovery.rc
```

### Import Libraries (`lib32/`)
```
pf_kernel32.lib   pf_user32.lib   pf_gdi32.lib
pf_advapi32.lib   pf_comctl32.lib pf_shell32.lib
pf_crypt32.lib    pf_shlwapi.lib  pf_rasapi32.lib
pf_comdlg32.lib   pf_ws2_32.lib
```

### Runtime (Project Root)
```
sqlite3mc.dll     (2 MB, 32-bit SQLite with cipher support)
```

## Regenerating Import Libraries

Only needed if you add a new system DLL dependency.

```powershell
# Run the generator (reads MASM32 .inc prototypes)
powershell -ExecutionPolicy Bypass -File lib32\genlibs.ps1

# Then rebuild each .lib from .def (in lib32 directory)
cd lib32
for %d in (kernel32 user32 gdi32 advapi32 comctl32 shell32 crypt32 shlwapi rasapi32 comdlg32 ws2_32) do (
    polib.exe /def:pf_%d.def /machine:x86 /out:pf_%d.lib
)
```

**Critical:** The `.def` files use `_Name@n=Name` decoration (e.g., `_CryptUnprotectData@28=CryptUnprotectData`). This is **required** for the Windows loader to resolve imports correctly. Do not use bare `Name@n=Name` form.

## Runtime Requirements

Place `sqlite3mc.dll` **next to the executable**:
```
Multi Password Recovery.exe
sqlite3mc.dll
```

The loader falls back to system `sqlite3.dll` if present, but `sqlite3mc.dll` provides cipher support for encrypted databases.

## Build Verification

```cmd
# Clean rebuild
del src\*.obj src\mpr.res
build.bat

# Should output:
# BUILD OK -> C:\path\Multi Password Recovery.exe

# Run
Multi Password Recovery.exe
```

## Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| `ml.exe` not found | MASM32 not at `C:\masm32` | Install MASM32 to `C:\masm32` or edit `build.bat` |
| `polink.exe` missing | Incomplete MASM32 install | Reinstall MASM32 (includes Pelle tools) |
| `pf_*.lib` missing | `lib32/` not populated | Run `genlibs.ps1` + `polib` loop above |
| `sqlite3mc.dll` missing at runtime | Not copied to exe directory | Copy from project root |
| Link error: unresolved external | Missing `.obj` in link line | Add module to `build.bat` FOR loop and link line |
| Duplicate `shlwapi.inc` warning | Multiple includes | Already fixed in `pv.inc` (single include) |

## Architecture Notes

- **32-bit only** (x86) — runs on WoW64 on 64-bit Windows
- **Subsystem: WINDOWS** — GUI application (no console)
- **Manifest:** Common Controls 6.0 (visual styles)
- **Linker:** Pelle Polink (not MS link.exe) — required for `pf_*.lib` format
- **Calling convention:** STDCALL throughout

## Adding a New Module

1. Create `newmodule.asm` in `src/`
2. Add `ScanNewModule proto` to `pv.inc`
3. Implement `ScanNewModule` proc calling `PV_Add` for each credential
4. Add `newmodule` to `build.bat` FOR loop
5. Add `.\newmodule.obj` to polink command line
5. Add `invoke ScanNewModule` to `ScanAll` in `winmods.asm`
6. Rebuild

## Output

| File | Size | Description |
|------|------|-------------|
| `Multi Password Recovery.exe` | ~600 KB | Main GUI executable |
| `sqlite3mc.dll` | ~2 MB | Runtime SQLite (must accompany exe) |

No installer, no registry, no .NET, no VC++ runtime — fully portable.