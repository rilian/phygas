---
id: PHY-007
title: Windows builds that use msvcrt.dll may silently drop cluster or target lines that have nan in a numeric column
type: bug
status: new
severity: medium
platforms: [windows, mingw]
area: io
next: Check whether the shipped Windows exe imports msvcrt.dll (objdump -p) and whether young_clusters.txt has nan outside columns 12-13
pattern:
aliases: [msvcrt NaN parse]
reported: 2026-10-21
source: audit 2026-10-21 (plan section 3.3, "A %lf NaN parse on msvcrt")
where: src/phygas.c:446-449 @ 0a3eda2
fixed_in:
---

## Symptom

None reported. If it happens: a Windows build reads fewer clusters than a Linux build of the same task, and every
row with those clusters is missing from the Windows result only. Nothing in stderr shows it: the BOINC app does not
print how many clusters or targets it read.

## Cause

Read at 0a3eda2; whether it is triggered is still to be confirmed.

* `ph_read_clusters` (`src/phygas.c:446-449`) reads 17 of the 19 columns with `%lf` and skips the line without a
  message when `sscanf` does not return 19. Only `rv` and `e_rv` are read as strings and compared with `"nan"`
  (451, `parse_num` at 434). `ph_read_targets` (469-471) and `parse_target` (`src/main_phygas_boinc.cpp:313-315`)
  do the same for targets.
* Microsoft's old `msvcrt.dll` `scanf` does not accept `nan` for `%lf`; glibc, UCRT and mingw-w64's own `scanf`
  (`__USE_MINGW_ANSI_STDIO`) do. `phygas.c` is C compiled with `-std=c99` (`scripts/build_boinc_all.sh:23`); in that
  mode the mingw-w64 headers do not switch to their own `scanf` by themselves, so `sscanf` is the C runtime's. The
  x64 and x86 builds use the distribution's `x86_64-w64-mingw32-gcc` / `i686-w64-mingw32-gcc`
  (`build_boinc_all.sh:41-42`, `build_gpu_server.sh:66-75`), which often defaults to `msvcrt.dll`. llvm-mingw
  (arm64) defaults to UCRT, and MSVC (CUDA) uses UCRT. `main_phygas_boinc.cpp` is C++; libstdc++ on mingw-w64 turns
  on mingw's own stdio, so `parse_target` is probably not affected (not verified).
* Can the inputs contain `nan` there? `scripts/prepare_clusters.py` checks or replaces every numeric column except
  `age50` and `age84` (columns 17 and 18): they are written as `nan` when Hunt & Reffert give `logAge16` but not
  `logAge50`/`logAge84` (68, 79-80). Those two columns are not used in the computation, but a `nan` there still
  makes the line fail to parse. `scripts/prepare_pulsars.py` writes `ra` and `dec` without a NaN check (58);
  `scripts/make_star_catalog.py` writes only numbers in the `%lf` columns. Whether the production files hold such a
  line is not known.

To confirm: (1) on the cluster file the server sends, count lines with `nan` outside columns 12-13:
`awk '{for(i=4;i<=19;i++) if(i!=12 && i!=13 && $i=="nan"){print; break}}' young_clusters.txt | wc -l`;
(2) `x86_64-w64-mingw32-objdump -p phygas_v1_windows_x86_64.exe | grep 'DLL Name'` shows `msvcrt.dll` or the UCRT
`api-ms-win-crt-*` DLLs; (3) if both hold, run one task on Windows and on Linux and compare.

## Fix

If confirmed: parse every numeric column through a helper that accepts `nan` (as `parse_num` does), or have
`prepare_clusters.py` write a number for the unused age columns; print the number of clusters and targets read to
stderr in the BOINC app, so that a mismatch is visible in task logs.

## Test

A cluster file with `nan` in column 17, read by a mingw-w64 build linked against `msvcrt.dll` (under Wine or on
Windows): the number of clusters read must equal the Linux build's.

## Log

- 2026-10-21: listed in the audit (plan section 3.3), at 0a3eda2.
- 2026-10-27: recorded from the 2026-10-21 audit; parsers and writers read at 0a3eda2. Not reproduced (no mingw-w64
  or Windows here); the C runtime of the shipped binaries and the content of the production cluster file are still
  to be checked.
