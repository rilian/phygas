---
id: PHY-010
title: The Windows CUDA build script needs a BOINC tree at a path on one developer's machine
type: bug
status: confirmed
severity: low
platforms: [windows, cuda, msvc]
area: build
next: In build_gpu_windows.bat, take BOINC, NVCC and VCVARS from environment variables when set, with today's values as defaults
pattern: PAT-021
aliases: [Windows CUDA hard-coded path]
reported: 2026-10-21
source: audit 2026-10-21 (plan section 3.3)
where: scripts/build_gpu_windows.bat:9 @ 0a3eda2
fixed_in:
---

## Symptom

`scripts\build_gpu_windows.bat` fails on any other Windows machine: `cl` and `nvcc` do not find the BOINC headers
and libraries, and the script stops with `exit /b 1`. Nobody else can rebuild the Windows CUDA app.

## Cause

Verified by reading at 0a3eda2. Line 9 sets `BOINC` unconditionally to a directory under one user's profile
(`C:\Users\<user>\Documents\...\boinc`; the user name is left out here). Lines 4 and 5 also fix the Visual Studio
2022 Build Tools `vcvars64.bat` and the CUDA v13.3 `nvcc.exe` at their default install paths; those exist on other
machines with the same versions. Only the version can be overridden (`PHYGAS_VERSION`, 6-7).

The other build scripts use the build server's layout (`/opt/bitboinc-src/boinc-<target>`, `/opt/llvm-mingw/bin`,
`/usr/bin/x86_64-w64-mingw32-*`), documented in their headers; the PAT-021 grep does not match them.

## Fix

Not fixed. Take `BOINC` (and `NVCC`, `VCVARS`) from the environment when set, with the current values as defaults
(`if "%BOINC_DIR%"=="" ...`), and document the variables in `AGENTS.md`. nostos's `build_gpu_windows.bat` has the
same line.

## Test

Run the script on a Windows machine with `BOINC_DIR` pointing to a BOINC tree built with `win_build` (x64 Release):
it must build `build\gpuwin\phygas_cuda_v2_windows_x86_64.exe`.

## Log

- 2026-10-21: listed in the audit (plan section 3.3), at 0a3eda2.
- 2026-10-27: recorded from the 2026-10-21 audit, verified at 0a3eda2 by reading the build scripts.
