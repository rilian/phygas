---
id: PHY-011
title: Pulsar work units are one target, so they have no checkpoint and no progress until the end
type: design-note
status: documented
severity: low
platforms: [all]
area: boinc-lifecycle
pattern:
aliases: [plan 6.2 design note]
reported: 2026-10-21
source: audit 2026-10-21 (plan section 6.2)
where: src/main_phygas_boinc.cpp:459-465 @ 0a3eda2
fixed_in:
---

## Symptom

A pulsar task shows 0 % for its whole run and starts again from the beginning after every quit (client exit,
reboot, preemption with "leave in memory" off).

## Cause

Read at 0a3eda2. Checkpoints and progress advance by whole targets: the CPU loop writes the checkpoint and calls
`boinc_fraction_done()` after a target (459-465); `gpu_pair` checkpoints in `emit` only when a target is finished
(`upto > written`, 261-268). README: a pulsar work unit is one pulsar with 1000 clones, about 10 minutes on a CPU.
With one target there is no checkpoint before the end and the progress jumps from 0 to 1.

This is the "no progress reporting at all" case of PAT-007; the resume case for star work units is PHY-017.

## Fix

None needed at the current size: a quit costs at most one task's run (about 10 minutes on a CPU; on a GPU much
less, commit b63a70a reports a 54-pair stage-2 task in 5 s instead of 12.8 min). If pulsar work units grow (more
clones, more clusters), checkpoint inside the target: the pairs of a target are run in cluster order with
deterministic seeds, so the number of finished pairs is a valid resume point, and progress can be reported per
pair.

## Test

Not applicable. A pulsar task on a CPU, suspended with "leave in memory" off and resumed, restarts at 0 %.

## Log

- 2026-10-21: noted as a design note in the audit (plan section 6.2), at 0a3eda2.
- 2026-10-27: recorded from the 2026-10-21 audit, verified at 0a3eda2 by reading `src/main_phygas_boinc.cpp`.
