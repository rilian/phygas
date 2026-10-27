---
id: PHY-001
title: Rows written after the last checkpoint are kept on resume and computed again, so out gets duplicate and torn lines
type: bug
status: confirmed
severity: high
platforms: [all]
area: checkpoint
next: On resume, cut out back to the rows of targets below the checkpoint (temp file, boinc_rename), drop a torn tail, write the end line only once
pattern: PAT-002
aliases: [P1]
reported: 2026-10-21
source: audit 2026-10-21 (plan sections 1 and 6.2)
where: src/main_phygas_boinc.cpp:413-421 @ 0a3eda2
fixed_in:
---

## Symptom

After a restart of a task (client quit, reboot, crash, power loss) the result file `out` holds the rows of some
targets twice, and sometimes one broken line where a cut row runs into the next row. From the reproduction below:

```
1141 C250 o 742.040 3.500 912.041140 C0 o 481.094 4.000 1036.70 30.77 0.0000 0.0000 49.69
```

A validator that compares two results line by line, or an assimilator that does not remove duplicates, sees extra
or broken rows. No volunteer report yet.

## Cause

Verified by reading `src/main_phygas_boinc.cpp` at 0a3eda2 and reproduced (Test).

* The resume position comes only from `phygas_checkpoint` (413-419). `out` is then opened with `"a"` (420) and is
  never cut back to what the checkpoint describes.
* CPU path (431-466): the rows of a target are printed as soon as it is done; the checkpoint is written only when
  `boinc_time_to_checkpoint()` returns 1 (by default every 60 s) or after the last target (459-464). At about 0.02 s
  per star (commit 8bb46cb) and about 290 bytes of output per star (README: 5.8 MB for 20,000 stars), some 0.8 MB of
  rows are printed between two checkpoints, so the stdio buffer is written to disk many times in between. Those rows
  are on disk at any kill, and on a client quit the BOINC library writes the rest with `fflush(NULL)`
  ([boinc-api.md](../shared/boinc-api.md#exit-paths)). After the restart the targets from the checkpoint on are
  computed and appended again. So on the CPU path nearly every restart duplicates rows.
* A buffer write usually ends inside a row. When the process ends without a flush (crash, power loss, a kill after
  a quit timeout) the file ends with a partial row, and the first appended row continues it on the same line. A quit
  that arrives inside an `fprintf` can do the same.
* GPU screen path (`gpu_screen`, 161-171): the rows of a batch are printed and flushed (168), then the checkpoint is
  written (169-170). A quit or kill between the two keeps the rows without the checkpoint that counts them. The
  window is short (host printing only), so this is rare, but one hit duplicates up to a whole batch
  (`PHYGAS_BATCH_STARS`, default 1024 stars).
* GPU pair path (`gpu_pair`, `emit` at 251-269): the same order, rows, `fflush` (262), checkpoint (263-264).
* End marker: `# done <nt>` (467) is appended without checking whether it is already there. If a run has
  written the final checkpoint (`nt`) and `# done` reached the disk, but the task did not end as finished, the
  restart reads `nt`, runs no target and appends a second `# done` line. Windows for this: a quit between the
  `fprintf` of `# done` (467) and `boinc_finish(0)` (469), where the library's `fflush(NULL)` writes the line (a few
  statements long); a crash, power cut or kill during the 2 s sleep of `boinc_finish()` before it writes
  `boinc_finish_called`. A quit during that sleep is not acted on, because the timer handler stops once `finishing`
  is set ([boinc-api.md](../shared/boinc-api.md#exit-paths)). Read from the code; not reproduced.
* Flush order is right: every path flushes `out` before it writes the checkpoint (CPU 460-461, GPU screen 168-169,
  GPU pair 262-263), so a checkpoint never counts rows that are still in the stdio buffer. A power cut can still lose
  data the OS had not written yet (no `fsync`), in either file.

What makes a clean fix possible: every row depends only on its seeds (`ph_seed(target, cluster)`, and `"lin"` /
`"track"` per target, `src/phygas.c:426-432`), not on when it was computed, so recomputed rows are byte-identical
copies (the reproduction confirms it).

## Fix

Not fixed. Proposed in the plan (P1): a checkpoint line `phygas_checkpoint 2 <targets_done> <lines_done>`, written
through a temporary file and `boinc_rename` (PHY-002). On resume, keep exactly the first `lines_done` complete lines
of `out` (copy them to a temporary file, rename it over `out`); if `out` has fewer, restart from 0. An old
one-number checkpoint means restart from 0 (question for the maintainers: or honour it when `out` is consistent with
it). Write `# done` only after checking that the kept part of `out` does not end with it already. One helper for
the CPU path and both GPU paths. planodia's `trim_output()` (PAT-002 reference fix) counts one line per item; phygas
writes zero or more rows per target (threshold output), so it needs the line count in the checkpoint, or the variant
that PAT-002 allows when each line carries its item id: rows carry the target id in the first column and come in
target order, so the resume can keep the leading rows whose target index (from `in`) is below the checkpoint's count
and drop the rest, including a torn tail.

Count lines, not bytes, or switch to binary mode first (PHY-006): text mode changes the byte counts on Windows.
For volunteers: results without duplicates; a restart still costs the work since the last checkpoint. With a new
checkpoint format, tasks that resume across the update with an old checkpoint restart from 0.

## Test

Reproduced 2026-10-27 on macOS arm64 (Apple clang 14): `main_phygas_boinc.cpp` (CPU build) linked against a
30-line stand-in for the BOINC API written for this test (not in the repository). Its `boinc_time_to_checkpoint()`
returns 1 on every 10th call; at the 15th `boinc_fraction_done()` it does what a client quit does (`fflush(NULL)`,
then `exit`). Input: 30 synthetic stars, 300 synthetic clusters, header `PHYGAS 1 screen 1e9 64 2 0` (every pair is
printed).

* Uninterrupted: 9,001 lines. Quit, then resume: checkpoint 10, 10,501 lines, 1,500 duplicate lines (targets 10-14).
* 200 stars, the process ended by SIGALRM after 2 s (no flush), then resumed: checkpoint 140, `out` ended in the
  middle of a row; final file 60,551 lines instead of 60,001, 549 duplicate lines and the torn line above.

No test exists in the repository. The harness described in PAT-002 (stub quit = `fflush(NULL)` + `exit` at random
points, then `cmp` with the uninterrupted output) would cover all three paths; the GPU paths need a GPU host.

## Log

- 2026-10-21: found in the audit (plan P1), at 0a3eda2.
- 2026-10-27: recorded from the 2026-10-21 audit, verified at 0a3eda2 by reading `src/main_phygas_boinc.cpp` and by
  the stub reproduction above.
- 2026-10-27: re-checked after PAT-002 was corrected (flush before checkpoint, end markers). Verdict unchanged:
  the flush order is right on all three paths; the end-marker case is restated with the corrected
  `boinc_finish()` facts (a quit during its sleep is ignored, so only a quit just before it or a crash/kill during
  it applies); the fix now says to write `# done` only when it is not there.
