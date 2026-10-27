---
id: PHY-014
title: make_star_catalog.py needs a nostos checkout next to this repository
type: design-note
status: documented
severity: info
platforms: [all]
area: data-pipeline
pattern:
aliases: [make_work_units.py location]
reported: 2026-10-21
source: audit 2026-10-21 (plan sections 3.3 and 8)
where: scripts/make_star_catalog.py:23-28 @ 0a3eda2
fixed_in:
---

## Symptom

`python scripts/make_star_catalog.py <gaia_csv> <out.txt.gz>` fails with `ModuleNotFoundError: make_work_units`
when there is no nostos checkout at `../nostos` next to this repository.

## Cause

Read at 0a3eda2. The script imports `zero_points` from `make_work_units` (28), from the directory given by
`--nostos-scripts`, by default `../../nostos/scripts` relative to the script (23-24), i.e. a sibling checkout named
`nostos`. The audit (2026-10-21) said that file was in neither repository. On 2026-10-27 it is in the nostos
repository: `scripts/make_work_units.py` (added in nostos 2389f5b; `zero_points()` since 13ba17b, 2026-10-08;
checked at nostos eb7029e). `zero_points(rows, "lindegren")` imports `numpy` and the `zero_point` package (the Gaia
DR3 parallax zero-point of Lindegren et al. 2021).

## Fix

None needed. Keep the shared code in nostos; run with `--nostos-scripts DIR` when the checkout is elsewhere. The
script writes `<out>.part` and renames it at the end (37, 71), so an interrupted run leaves no partial catalogue.

## Test

With nostos checked out next to phygas and `numpy` and `zero_point` installed:
`python3 scripts/make_star_catalog.py gaia.csv stars.txt.gz --max-rows 1000`.

## Log

- 2026-10-21: the audit asked where `make_work_units.py` lives (plan section 8).
- 2026-10-27: recorded; the file exists in nostos (eb7029e), checked with `git log` there.
