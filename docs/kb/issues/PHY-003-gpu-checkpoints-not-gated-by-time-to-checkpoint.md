---
id: PHY-003
title: The GPU paths write the checkpoint after every batch without boinc_time_to_checkpoint() and report each one
type: bug
status: confirmed
severity: low
platforms: [cuda, opencl]
area: checkpoint
next: In gpu_screen and gpu_pair, write the checkpoint only when boinc_time_to_checkpoint() returned 1 (kept in a variable) and after the last target
pattern: PAT-004
aliases: [P2]
reported: 2026-10-21
source: audit 2026-10-21 (plan section 6.2)
where: src/main_phygas_boinc.cpp:168-172 @ 0a3eda2
fixed_in:
---

## Symptom

None visible to volunteers in normal runs. The GPU app ignores the "checkpoint at most every N seconds" preference,
and its checkpoint writes are not protected from a quit.

## Cause

Verified by reading `src/main_phygas_boinc.cpp` at 0a3eda2.

* `gpu_screen` writes `phygas_checkpoint` after every batch and calls `boinc_checkpoint_completed()` each time
  (168-172). A batch is `PHYGAS_BATCH_STARS` stars (default 1024, halved until the GPU buffers fit, 91-97). Commit
  8bb46cb reports 20,000 stars in 12 s on an RTX 5070 Ti: about 20 checkpoints in 12 s.
* `gpu_pair` writes it in `emit` whenever a flush completes targets (261-268); a flush happens after at least
  `PHYGAS_BATCH_CLONES` clones (default 262144, 192 and 292) or when the device is full (277-280).
* Neither asks `boinc_time_to_checkpoint()`. So the write is outside a critical section and, being in place
  (PHY-002), a quit inside it can leave an empty checkpoint (restart from 0); the volunteer's interval is ignored
  (the file is a few bytes, so disk wear does not matter); `boinc_checkpoint_completed()` without a preceding 1
  resets the client's checkpoint bookkeeping at the application's rhythm.

PAT-001 was checked at the same time: the critical-section counter is clamped at 0, so these extra
`boinc_checkpoint_completed()` calls do not leave a critical section open. The CPU path,
`if (boinc_time_to_checkpoint() || j + 1 == nt)` (459), calls `boinc_time_to_checkpoint()` exactly once per target
as the first operand, and every 1 is followed by `boinc_checkpoint_completed()` on the only path (463); when only
`j + 1 == nt` is true, the final checkpoint is reported without a 1, which is harmless. The GPU build never runs the
CPU loop (`start = nt`, 429).

## Fix

Not fixed. Plan P2: ask `boinc_time_to_checkpoint()` once per batch (keep the result in a variable), write the
checkpoint when it returned 1 and after the last target, and call `boinc_checkpoint_completed()` only for the writes
the timer asked for. Use the helper of PHY-002, shared with the CPU path. The rows printed for a batch must stay
consistent with the checkpoint (PHY-001).

## Test

Stub BOINC library that logs every `boinc_time_to_checkpoint()` result and every checkpoint write; with the timer
never firing, only the final checkpoint may appear. Needs a GPU host, or a stub GPU backend (`pg_*` functions that
run the CPU code).

## Log

- 2026-10-21: found in the audit (plan P2), at 0a3eda2.
- 2026-10-27: recorded from the 2026-10-21 audit, verified at 0a3eda2 by reading `src/main_phygas_boinc.cpp`.
