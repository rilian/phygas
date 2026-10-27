---
id: PHY-002
title: phygas_checkpoint is rewritten in place, its writes are not checked, and the value read back is not range-checked
type: bug
status: confirmed
severity: low
platforms: [all]
area: checkpoint
next: Write phygas_checkpoint through one helper: temp file, checked fprintf/fclose, boinc_rename, report only on success; reject values outside 0..nt
pattern: PAT-003
aliases: [P1, P3]
reported: 2026-10-21
source: audit 2026-10-21 (plan section 6.2)
where: src/main_phygas_boinc.cpp:461-463 @ 0a3eda2
fixed_in:
---

## Symptom

None reported. What can happen:

* a kill or quit between the truncation and the write, or a full disk, leaves an empty `phygas_checkpoint`; the
  restart then begins at target 0 and truncates `out` (finished work lost, the result stays consistent);
* the client is told a checkpoint was written when it was not;
* a corrupted value: a negative one makes the loop read `tg[]` before its start (crash or garbage), a value above
  the number of targets ends the task as a success with whatever rows `out` holds.

## Cause

Verified by reading `src/main_phygas_boinc.cpp` at 0a3eda2. The checkpoint is written in three places, the same
way each time:

| Path | Lines |
|---|---|
| GPU screen, after every batch | 169-171 |
| GPU pair, in `emit` | 263-265 |
| CPU, timer or last target | 461-463 |

* (a) `boinc_fopen("phygas_checkpoint", "w")` truncates the real file; there is no temporary file and no rename.
* (b) the results of `fprintf` and `fclose` are not checked.
* (d) `boinc_checkpoint_completed()` is called even when `boinc_fopen` returned NULL.
* Reading (414-419): `fscanf(cp, "%d", &start)` only checks that a number was read. There is no `0 <= start <= nt`
  check. With `start < 0`, `out` is opened with `"w"` (420) and the CPU loop starts at `tg[start]` (431); the GPU
  paths index `tg[j0 + s]` the same way. With `start > nt` no target runs, `# done <nt>` is appended and the task
  ends with `boinc_finish(0)`.

## Fix

Not fixed. One helper for all three paths (plan P1 and P3): the `write_file_atomic()` of PAT-003 (temporary file,
checked `fwrite`/`fflush`/`fclose`, `boinc_rename`), `boinc_checkpoint_completed()` only on success (and
`boinc_end_critical_section()` when the timer asked and the write failed, PAT-001), and on reading: a value outside
`0..nt`, or a line that does not parse, means restart from 0 with `out` truncated. Do it together with PHY-001,
which changes the checkpoint content.

## Test

PAT-003 "How to test": make the write fail (read-only slot directory, or `/dev/full` on Linux) and check that the old
checkpoint is unchanged and nothing was reported. Put `-5` and `nt + 10` into `phygas_checkpoint` and check that the
task restarts from 0.

## Log

- 2026-10-21: found in the audit (plan P1 for the write, P3 for the range check), at 0a3eda2.
- 2026-10-27: recorded from the 2026-10-21 audit, verified at 0a3eda2 by reading `src/main_phygas_boinc.cpp`.
