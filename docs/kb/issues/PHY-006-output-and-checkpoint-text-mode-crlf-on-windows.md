---
id: PHY-006
title: out and phygas_checkpoint are written in text mode, so the Windows builds write CRLF line ends
type: bug
status: confirmed
severity: low
platforms: [windows]
area: io
next: Open out with "wb"/"ab" and the checkpoint with "wb"; before release confirm the validator accepts LF line ends from Windows hosts
pattern: PAT-019
aliases: []
reported: 2026-10-21
source: audit 2026-10-21 (plan section 3.3); commit 88ed006 message
where: src/main_phygas_boinc.cpp:420 @ 0a3eda2
fixed_in:
---

## Symptom

Results from Windows hosts end every line with `\r\n`; results from every other platform with `\n`. The commit
message of 88ed006 says so: output "identical on x86_64, i686 (qemu) and Windows x64 (wine) apart from CRLF line
ends". A validator or checksum that compares bytes sees Windows results as different.

## Cause

Verified by reading `src/main_phygas_boinc.cpp` at 0a3eda2: `out` is opened with `"a"` or `"w"` (420) and
`phygas_checkpoint` with `"w"` (169, 263, 461): text mode. The C runtime on Windows turns every `\n` into `\r\n` in
text mode (MinGW and MSVC alike). Windows builds: `scripts/build_boinc_all.sh:41-43` (mingw-w64 x64 and x86,
llvm-mingw arm64), `scripts/build_gpu_server.sh:64-76` (OpenCL, mingw-w64), `scripts/build_gpu_windows.bat` (CUDA,
MSVC).

`phygas_checkpoint` is private to the application (one number read back with `fscanf`), so its CRLF does no harm
as long as nothing trims or compares it by bytes; the result file `out` is what matters.

The readers are not affected: inputs are opened with `"r"` (`main_phygas_boinc.cpp:372, 414`,
`src/phygas.c:437, 461`), `sscanf` `%s` stops at `\r`, and `parse_target` strips `\r` from the `clusters=` list
(324). A CRLF input file parses.

## Fix

Not fixed. Open `out` with `"wb"`/`"ab"` and the checkpoint (and the temporary files of the PHY-002 fix) with
`"wb"`. That changes the bytes of Windows results (no CR any more): ask the maintainers whether the validator or
assimilator expects CRLF from Windows hosts, and deploy it as a new app version. A trim on resume (PHY-001) needs
this, or must count lines instead of bytes.

## Test

Build with mingw-w64, run a small work unit on Windows (or under Wine), then `grep -c $'\r' out` must be 0 and `cmp`
with the Linux output must succeed.

## Log

- 2026-10-21: listed in the audit (plan section 3.3, "Text-mode output means CRLF on Windows"), at 0a3eda2.
- 2026-10-27: recorded from the 2026-10-21 audit, verified at 0a3eda2 by reading `src/main_phygas_boinc.cpp` and
  the build scripts; CRLF itself is confirmed by the commit message of 88ed006 (not rerun here).
- 2026-10-27: re-checked after PAT-019 was corrected (private text files): verdict unchanged, because `out` is the
  result file; noted that the checkpoint alone would not count.
