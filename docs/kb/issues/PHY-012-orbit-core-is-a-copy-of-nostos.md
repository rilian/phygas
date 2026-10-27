---
id: PHY-012
title: The orbit core is a copy of the nostos files and must be kept identical, also with the GPU port
type: design-note
status: documented
severity: info
platforms: [all]
area: science
pattern:
aliases: [Nostos core]
reported: 2026-10-21
source: audit 2026-10-21 (plan section 3.3, nostos part)
where: src/potential.c:1 @ 0a3eda2
fixed_in:
---

## Symptom

A fix made in one repository's copy of the coordinate, potential or Monte Carlo code does not reach the other, and
the two applications (or phygas's CPU and GPU paths) silently use different physics.

## Cause

Checked 2026-10-27 with `cmp`: `src/coords.c`, `coords.h`, `potential.c`, `potential.h`, `montecarlo.c`,
`montecarlo.h`, `integrator.h`, `vec3.h` and `units.h` are byte-identical to the same files in the nostos repository
(nostos eb7029e). The plan (2026-10-21) said the core "currently lives only in phygas/src"; that is not the case at
these commits. The tests of the core (`tests/test_coords.c`, `test_kepler.c`, `test_milkyway_*.c`,
`test_montecarlo.c`) and `NOTES_method_selection.md` are in nostos only.

`integrator.h` is not used by phygas: `src/phygas.c` has its own leapfrog with one force evaluation per step
(`body_step`, 91-96). The GPU device code ports `milkyway_accel` (`pg_accel`, `src/phygas_kernel_body.h:17-50`) and
the RNG and Gaussian draw of `montecarlo.c`/`.h` (166-179) operation by operation, with the potential's parameters
passed from `milkyway_potential_default` (`src/main_phygas_boinc.cpp:79-80, 190-191`).

## Fix

None; kept as a copy on purpose. The files are registered as `shared-code` in `docs/kb/kb.conf`, so
`python3 docs/kb/shared/kb.py status ../nostos` reports a drift. A change to one of them must be copied to nostos in
the same change, and a change to `potential.c` or `montecarlo.*` must also be ported to
`src/phygas_kernel_body.h` (then `python3 scripts/embed_kernel.py`), or CPU and GPU results diverge.

## Test

`kb.py status ../nostos` shows no drift; the CPU and GPU outputs of one work unit are identical (commits 8bb46cb and
b63a70a describe how this was checked).

## Log

- 2026-10-21: the audit noted the core's location (plan section 3.3).
- 2026-10-27: recorded; copies compared with nostos eb7029e (identical); use of `integrator.h` checked with `grep`.
