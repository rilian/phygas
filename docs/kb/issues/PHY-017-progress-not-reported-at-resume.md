---
id: PHY-017
title: Progress is not reported at resume until the first new target or GPU batch is done
type: bug
status: not-a-bug
severity: info
platforms: [all]
area: boinc-lifecycle
pattern: PAT-007
aliases: []
reported: 2026-10-27
source: pattern ledger check, 2026-10-27
where: src/main_phygas_boinc.cpp:413-419 @ 0a3eda2
fixed_in:
---

## Symptom

After a restart the client shows 0 % until the first new target (CPU) or batch (GPU) is finished.

## Cause

Verified by reading at 0a3eda2. Nothing reports the resumed position after the checkpoint is read (413-419). The
first `boinc_fraction_done()` comes after the first new target (CPU, 465) or batch (`gpu_screen` 172, `gpu_pair`
266). For star work units the gap is short: one target on the CPU (about 0.02 s in screen mode, commit 8bb46cb),
one batch of up to 1024 stars on the GPU (seconds). Pulsar work units have no resume at all (PHY-011).

## Fix

None needed under the corrected PAT-007 (only items, or a time to the first report, longer than about a minute
count). Estimated time from a restart to the first report: start-up precomputation of the clusters (300 synthetic
clusters took 0.1 s on an Apple M1 in `phygas_screen`, so about 1.5-3 s for 4,087 clusters) plus one target (about
0.02 s in screen mode; a few pairs of 1000 clones, well under a second, in stage-2 pair mode) or one GPU batch
(about 0.6 s on an RTX 5070 Ti per commit 8bb46cb; a GPU ten times slower in fp64 still stays under a minute). If
cheap, `boinc_fraction_done((double)start / nt)` right after the resume position is known still costs nothing.

## Test

Stub BOINC library that logs `boinc_fraction_done()`: after a resume the first value must equal start / nt.

## Log

- 2026-10-27: found while checking PAT-007 at 0a3eda2.
- 2026-10-27: re-checked after PAT-007 was corrected; the first report after a resume comes within seconds (estimate
  above), so status not-a-bug and the ledger row not-affected.
