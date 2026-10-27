---
id: PHY-005
title: Results may differ in the last digits across platforms and backends (FMA contraction, libm, GPU built-ins)
type: bug
status: new
severity: medium
platforms: [aarch64, windows, macos, android, cuda, opencl]
area: determinism
next: Run one real work unit through CLI builds on Linux x86_64 and aarch64, Windows x64 and arm64, macOS arm64; diff; ask what the validator compares
pattern: PAT-018
aliases: [P5]
reported: 2026-10-21
source: audit 2026-10-21 (plan sections 1, 3.3 and 6.2)
where: scripts/build_boinc_all.sh:23 @ 0a3eda2
fixed_in:
---

## Symptom

None reported. What has been compared, according to the commit messages:

* 88ed006: "Output identical on x86_64, i686 (qemu) and Windows x64 (wine) apart from CRLF line ends". Windows was
  run only under Wine, which has its own C runtime, not Microsoft's.
* 8bb46cb: GPU output byte-identical to the CPU app on a full task with CUDA (Windows, Linux) and OpenCL (Windows,
  NVIDIA).
* b63a70a: pair mode on the GPU byte-identical on 2 pulsars x 4,080 clusters x 100 clones (Windows and Linux CUDA) and
  on a 54-pair stage-2 task.

Not compared: Linux aarch64, armhf, ppc64le, riscv64; Windows on real Windows and Windows arm64; macOS; Android;
AMD OpenCL.

## Cause

Read at 0a3eda2; what is still open is whether it happens with real work units.

* FMA contraction. The C core is compiled with `-std=c99` (`scripts/build_boinc_all.sh:23`,
  `scripts/build_gpu_server.sh:33, 46, 68`); GCC does not contract `a*b+c` in ISO C modes, so the GCC builds (all
  Linux targets, Windows x64 and x86) do not. MSVC (`build_gpu_windows.bat:14`, x64 without `/arch:AVX2`) does not.
  The Windows arm64 build uses llvm-mingw clang (`build_boinc_all.sh:43`); recent Clang contracts within one
  expression by default (`-ffp-contract=on`) whatever the `-std`, and the llvm-mingw version is not pinned. macOS
  and Android are built on other hosts (`build_boinc_all.sh:4-5`), presumably with clang. `main_phygas_boinc.cpp` is
  compiled as GNU C++ but does no result arithmetic of the form `a*b+c`. GPU: `nvcc --fmad=false`
  (`build_gpu_server.sh:34`, `build_gpu_windows.bat:15`) and `#pragma OPENCL FP_CONTRACT OFF`
  (`scripts/embed_kernel.py:9`). So the comment at `src/phygas_cuda.cu:1-2` ("the CPU builds do not contract a*b+c
  into FMAs") holds for the GCC and MSVC builds, not necessarily for clang builds.
* libm. The core calls `pow` (`src/potential.c:57, 71`), `exp` (`src/phygas.c:56`), `log` and `cos`
  (`src/montecarlo.c:15-16`), `sin`, `cos`, `asin`, `atan2` (`src/coords.c`) and `sqrt`. Their results differ in the
  last bit between glibc (static Linux builds), msvcrt or UCRT (Windows), Apple's libm, bionic, and the CUDA and
  OpenCL built-ins used by the device code (`src/phygas_kernel_body.h`: `pow` at 23 and 45, `log` and `cos` at
  177-178). glibc on x86_64 also chooses FMA variants of some of these functions at run time on CPUs that have FMA
  (not measured here).
* Thresholds. A row is kept only if `chi2 < chi2_max` and `p_r50 >= p_min` (`src/main_phygas_boinc.cpp:136, 163,
  438, 441`), and `ph_prefilter` decides which pairs are run in pair mode. A last-bit difference near one of these
  limits adds or drops a whole row.

Measured 2026-10-27 on an Apple M1 (Apple clang 14.0.0, one libm), synthetic input (300 clusters x 200 stars in
screen mode with `chi2_max` 1e9 so every pair is printed; 3 pulsars and 10 stars in pair mode): the CLIs built with
`-ffp-contract=on` and with `-ffp-contract=off`. The raw doubles differ (`chi2` in 59,041 of 60,000 pairs, by up to
7.8e-10 relative; `sig_pc` up to 4.9e-11; `t`, `d_nom_pc`, `vrel` up to 9e-14; `p_r50` and `p_rtot` never), but the
printed rows in the BOINC app's formats were byte-identical (60,000 screen rows, 1,631 pair rows). So contraction
alone rarely changes a printed row; a rare change at a rounding or threshold boundary remains possible. This Apple
clang does not contract by default (0 FMA instructions in the core at `-O2`; 193 in `phygas.c` with
`-ffp-contract=on`). libm differences between platforms were not measured.

The validator is not in this repository. README says "validator passes" for a full task; `src/montecarlo.h:9-11`
speaks of quorum agreement between replicas. Unknown: does it compare bytes, lines, or numbers with a tolerance, and
do quorums mix platforms, or CPU and GPU app versions (plan section 8, "Validator tolerance")?

## Fix

Measure first (PAT-018, plan P5): build the CLI, or the BOINC app against a stub, for Linux x86_64 and aarch64 (gcc,
Docker), Windows x64 and arm64 (on Windows, not Wine), macOS arm64 (clang), run the same real work unit and diff.
Add `-ffp-contract=off` to the clang builds only if it removes measured differences. libm differences could only be
removed with own implementations of `pow`, `exp`, `log`, `sin`, `cos`; the plan's `pow(x,1.5)` to `x*sqrt(x)` would
change science results and is not proposed without the maintainers. Correct the comment in `phygas_cuda.cu` once the
answer is known.

## Test

None exists. A cross-platform diff script over a few small real inputs, run whenever compiler flags or math code
change. The local measurement above: build `phygas_screen` and `phygas_cli` twice with
`cc -O2 -std=c99 -ffp-contract=on|off -I src src/phygas.c src/coords.c src/potential.c src/montecarlo.c
src/phygas_screen.c -lm` and `cmp` the outputs.

## Log

- 2026-10-21: raised in the audit (plan P5, "measure first"), at 0a3eda2.
- 2026-10-27: recorded from the 2026-10-21 audit; build flags and math calls verified at 0a3eda2 by reading the
  scripts and sources; contraction effect measured on Apple M1 with synthetic input (above). Status stays new: no
  cross-platform measurement with real work units, validator unknown.
