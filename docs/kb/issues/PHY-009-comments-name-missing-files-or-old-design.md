---
id: PHY-009
title: Comments name files that do not exist in this repository or describe an older design
type: bug
status: confirmed
severity: low
platforms: [all]
area: docs
next: Fix the comments at phygas_gpu.h:2 and main_phygas_boinc.cpp:9 and 30-31; change cl_dynload.h and core comments only in all copies at once
pattern:
aliases: [stale comments]
reported: 2026-10-21
source: audit 2026-10-21 (plan section 3.3)
where: src/phygas_gpu.h:2 @ 0a3eda2
fixed_in:
---

## Symptom

A reader (or a coding agent) looks for files and functions that are not here, or believes the GPU app runs pair
mode on the CPU.

## Cause

Verified at 0a3eda2 by `grep` for every file name in comments and by reading the history of
`src/main_phygas_boinc.cpp`.

Files of phygas itself:

* `src/phygas_gpu.h:2`: "used by main_phygas_gpu_boinc.cpp". No such file; the GPU app is `main_phygas_boinc.cpp`
  built with `-DPG_GPU` (nostos has a `main_nostos_gpu_boinc.cpp`, which is probably where the line came from).
* `src/main_phygas_boinc.cpp:30-31`: "pair mode (pulsars, stage 2) is the CPU code below". Out of date since
  b63a70a (2026-10-10), which added `gpu_pair` (187-301) without changing this comment; in a GPU build both modes run
  on the GPU and the CPU loop is skipped (429).
* `src/main_phygas_boinc.cpp:9`: "the same columns as phygas_screen / phygas_cli". The CLIs print two more columns,
  `age16 age84` (`src/phygas_screen.c:60-61`, `src/phygas_cli.c:52-54`).

Shared copies (identical in nostos, see `kb.conf`):

* `src/cl_dynload.h:2, 15, 18`: "opencl_search.c", "gpu_search_init()". Not in phygas; the user here is
  `src/phygas_opencl.c` and `pg_init()`.
* The orbit core points to nostos files: `tests/test_coords.c`, `tests/test_horizons_crosscheck.c`,
  `tests/test_coords_astropy_crosscheck.c`, `tests/test_kepler.c`, `tests/test_milkyway_allen_santillan.c`,
  `tests/test_milkyway_galpy.c`, `orbital_elements.c` and `NOTES_method_selection.md` (`src/coords.h:5-12, 57, 63`;
  `src/coords.c:27-28, 89`; `src/integrator.h:6`; `src/potential.h:47, 64, 70`; `src/potential.c:121`;
  `src/montecarlo.h:4`; `src/units.h:46`). They exist in the nostos repository (checked at nostos eb7029e).

## Fix

Not fixed. Correct the three comments in phygas's own files. For the shared copies, either change them in every copy
at once (nostos too; `kb.py status` shows the drift otherwise) or leave them: `AGENTS.md` now says that the core's
tests and notes live in nostos.

## Test

`grep -rn 'main_phygas_gpu_boinc\|opencl_search\|gpu_search_init' src` finds nothing after the fix.

## Log

- 2026-10-21: listed in the audit (plan section 3.3), at 0a3eda2.
- 2026-10-27: recorded from the 2026-10-21 audit, verified at 0a3eda2; the `main_phygas_boinc.cpp:9` and core
  references were added on this check.
