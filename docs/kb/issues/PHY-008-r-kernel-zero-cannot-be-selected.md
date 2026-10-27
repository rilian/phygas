---
id: PHY-008
title: r_kernel 0 (use the total cluster radius) cannot be selected in a BOINC work unit
type: bug
status: confirmed
severity: low
platforms: [all]
area: science
next: In main_phygas_boinc.cpp, use r_kernel from the header whenever sscanf read it (0 = total radius), 2 only when absent; reject negative values
pattern:
aliases: [r_kernel 0]
reported: 2026-10-21
source: audit 2026-10-21 (plan section 3.3)
where: src/main_phygas_boinc.cpp:405 @ 0a3eda2
fixed_in:
---

## Symptom

A screen work unit whose header asks for `r_kernel` 0, for example `PHYGAS 1 screen 25 64 0 0.001`, runs with a
kernel radius of 2 x r50 instead of the total radius, without any message. `phygas_screen` with the same
parameters uses the total radius, so the CLI and the BOINC app give different rows.

## Cause

Verified by reading at 0a3eda2. `src/main_phygas_boinc.cpp:405`: `ph_screen_r = p3 > 0 ? p3 : 2.0;`. The
documented meaning of 0 is "use rtot" (`src/phygas.h:124`, used at `src/phygas.c:363`; `src/phygas_screen.c:2`
documents it and sets `ph_screen_r` directly at 23). In the BOINC app 0, a negative value and a missing value all
become 2. The GPU path uses the same `ph_screen_r` (`main_phygas_boinc.cpp:73`), so CPU and GPU agree with each
other.

## Fix

Not fixed. Use the count returned by the header `sscanf` (377): when `r_kernel` is present take it as given (0 =
total radius), use 2 only when it is absent, and reject a negative value with `fail("bad header")`. Work units that
pass `r_kernel` > 0 give the same rows as before.

## Test

The header `PHYGAS 1 screen 25 64 0 0.001` must give the same rows as
`phygas_screen clusters stars 25 64 0` filtered to `p_r50 >= 0.001` (without its two age columns).

## Log

- 2026-10-21: listed in the audit (plan section 3.3), at 0a3eda2.
- 2026-10-27: recorded from the 2026-10-21 audit, verified at 0a3eda2 by reading the files above.
