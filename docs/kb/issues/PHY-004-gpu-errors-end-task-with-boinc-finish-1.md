---
id: PHY-004
title: GPU errors end the task with boinc_finish(1) instead of a temporary exit
type: question
status: new
severity: medium
platforms: [cuda, opencl]
area: gpu-runtime
next: Decide: may GPU start-up and runtime errors end in boinc_temporary_exit() with a retry cap, instead of boinc_finish(1)?
pattern: PAT-013
aliases: [P4]
reported: 2026-10-21
source: audit 2026-10-21 (plan sections 6.2 and 8)
where: src/main_phygas_boinc.cpp:23-27 @ 0a3eda2
fixed_in:
---

## Symptom

A CUDA or OpenCL error during a task (launch failure, out of memory, driver reset, device busy or gone after a driver
update) ends the task as a computation error. The stderr shows a backend line such as
`phygas_cuda: cudaDeviceSynchronize() failed: ...` or `phygas_opencl: chi2 kernel failed (-5)`, then
`GPU screening failed`, `GPU pair run failed` or `no usable GPU`. The work done so far and its credit are lost and
the host's error count goes up.

## Cause

Verified by reading `src/main_phygas_boinc.cpp`, `src/phygas_cuda.cu` and `src/phygas_opencl.c` at 0a3eda2.
`fail()` (23-27) prints the message and calls `boinc_finish(1)`. GPU error exits, in the classes of PAT-013:

* (1) no usable device or start-up failure: `pg_init()` fails, then `fail("no usable GPU")` (357). This covers no
  OpenCL library, no matching platform, a kernel build error, no CUDA device or a driver that is too old.
* (2) error during the computation: any `pg_*` call in `gpu_screen` or `gpu_pair` returns non-zero, then
  `fail("GPU screening failed")` (425) or `fail("GPU pair run failed")` (427). "not enough GPU memory" (95, 204)
  ends the same way, although another program holding GPU memory can cause it.
* (3) deterministic: a device without fp64 (`phygas_opencl.c:67`). That one should stay an error, or better, never
  be sent there (plan class).

Checkpoints exist per batch (PHY-003), so a temporary exit would resume from the last batch.

## Fix

Question for the maintainers (plan section 8, "Temporary exit on GPU errors", the same question for kampylos and
planodia): should (1) and (2) end with `boinc_temporary_exit()` and a retry counter in the slot directory (about
five exits without progress, then `boinc_finish(1)`), as PAT-013 describes? Until the answer, nothing changes. If
yes, fix PHY-001 and PHY-002 first, so that the resumed output is consistent.

## Test

A stub GPU backend (or an environment switch in a test build) that makes the Nth `pg_*` call fail: the run must end
with a temporary exit and an intact checkpoint; repeated failures without progress must end with `boinc_finish(1)`.

## Log

- 2026-10-21: found in the audit (plan P4), at 0a3eda2.
- 2026-10-27: recorded from the 2026-10-21 audit, verified at 0a3eda2 by reading the files above. Waiting for the
  maintainers' decision.
