---
id: PHY-015
title: A checkpoint that exists but cannot be opened restarts the task from 0 and truncates out
type: bug
status: confirmed
severity: low
platforms: [all]
area: checkpoint
next: On resume, treat only !boinc_file_exists("phygas_checkpoint") as a fresh start; any other open failure -> boinc_temporary_exit()
pattern: PAT-005
aliases: []
reported: 2026-10-27
source: pattern ledger check, 2026-10-27
where: src/main_phygas_boinc.cpp:414-420 @ 0a3eda2
fixed_in:
---

## Symptom

None reported. After a restart the task starts again at target 0 and its finished rows are thrown away, although
`phygas_checkpoint` was there.

## Cause

Verified by reading at 0a3eda2. `boinc_fopen("phygas_checkpoint", "r")` (414) returning NULL leaves `start` at 0,
and `out` is then opened with `"w"` (420), which truncates it. NULL means "does not exist" or "could not be opened"
(on Windows after 5 s of retries: antivirus, indexer, backup; EACCES; EMFILE), see
[boinc-api.md](../shared/boinc-api.md#files). A checkpoint that does not parse (416) also gives start 0; in both
cases `out` is truncated to match, so the result stays correct and only work is lost.

## Fix

Not fixed. As in PAT-005: only `!boinc_file_exists("phygas_checkpoint")` means a fresh start; any other open failure
ends with `boinc_temporary_exit()`, leaving `out` alone. Part of the checkpoint helper of PHY-002.

## Test

On Unix: run until a checkpoint, `chmod 000 phygas_checkpoint`, restart: `out` must not be truncated and the task
must ask to be retried.

## Log

- 2026-10-27: found while checking PAT-005 at 0a3eda2.
