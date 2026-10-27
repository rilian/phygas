---
id: PHY-016
title: Write errors on out are not checked before boinc_finish(0)
type: bug
status: confirmed
severity: low
platforms: [all]
area: io
next: Before boinc_finish(0), check fflush, ferror and fclose of out; on failure print the error and end with boinc_finish(1)
pattern: PAT-006
aliases: []
reported: 2026-10-27
source: pattern ledger check, 2026-10-27
where: src/main_phygas_boinc.cpp:467-469 @ 0a3eda2
fixed_in:
---

## Symptom

None reported. On a full disk or a quota the task can upload a truncated `out` as a success.

## Cause

Verified by reading at 0a3eda2. No `fprintf` result is checked; `fflush(out)` (168, 262, 460) and `fclose(out)`
(468) results are ignored and `ferror(out)` is never called; then `boinc_finish(0)` (469). Whether the validator
rejects such a file depends on what it checks (for example that the last line is `# done <nt>`); the validator is
not in this repository.

## Fix

Not fixed. Before `boinc_finish(0)`: `if (fflush(out) != 0 || ferror(out) || fclose(out) != 0)` print the error and
end with `boinc_finish(1)` (or a temporary exit once the resume can repair `out`, PHY-001).

## Test

On Linux: run a standalone task with `out` a symbolic link to `/dev/full`; the run must not end with status 0.

## Log

- 2026-10-27: found while checking PAT-006 at 0a3eda2.
