# AGENTS.md: phygas

Orientation for coding agents and new contributors. The science is described in [README.md](README.md). Line
numbers below refer to commit 0a3eda2.

## What one work unit is

A task gets two input files and writes one. `clusters` is the young-cluster list (4,087 open clusters and moving
groups, 19 columns, written by `scripts/prepare_clusters.py`); it is the same for every task and stays on the
volunteer's machine (sticky). `in` starts with a header line, `PHYGAS 1 screen <chi2_max> <n_samp> <r_kernel>
<p_min>` or `PHYGAS 1 pair <n_clones> <dt_myr>`, followed by one target per line (a Gaia star or a pulsar, the
`PhTarget` format of `src/phygas.h`; in pair mode a line may end with `clusters=NAME1,NAME2,...`). Every target and
every cluster it can reach are traced back through an Allen & Santillan (1991) Milky Way potential, both drawn from
their measurement errors. Screen mode (stage 1 for stars, about 20,000 stars per task) uses the linear Gaussian
screening; pair mode (one pulsar per task, or stage 2 for stars with their listed clusters) runs the two-sided Monte
Carlo with `n_clones` clones. `out` gets one line per kept (target, cluster) pair and ends with `# done <n_targets>`.

## Layout

| Path | What it holds |
|---|---|
| `src/main_phygas_boinc.cpp` | The BOINC app: input parsing, checkpoint, CPU loop. With `-DPG_GPU` also `gpu_screen` and `gpu_pair` (CUDA with `-DPG_CUDA`, OpenCL with `-DPG_OPENCL`) |
| `src/phygas.h`, `src/phygas.c` | Core: draws with radial-velocity priors, leapfrog (`body_step`), tracks and prefilter, `ph_pair` (draw, clone, reduce), linear screening (`ph_lin_*`, `ph_screen_*`), `ph_seed`, file readers |
| `src/phygas_cli.c` | CLI, pair mode over a target file (pulsars, validation) |
| `src/phygas_screen.c` | CLI, stage-1 screening; `control` mode for the chance-alignment runs |
| `src/phygas_gpu.h` | Interface of the two GPU backends (`pg_*`), `pg_now`, `pg_next_slice` (launch sizing) |
| `src/phygas_kernel_body.h` | Device code shared by CUDA and OpenCL: operation-by-operation ports of `lin_integrate`, `ph_screen_chi2`, `ph_screen_finish`, `ph_pair_clone`, `milkyway_accel` and the RNG |
| `src/phygas_kernel_cl.inc` | Generated from `phygas_kernel_body.h` by `scripts/embed_kernel.py`; do not edit |
| `src/phygas_cuda.cu` | CUDA backend |
| `src/phygas_opencl.c` | OpenCL backend: platform and device choice, fp64 check, kernel build, work-group size |
| `src/cl_dynload.c`, `.h` | Loads the OpenCL library at run time; a copy shared with nostos and planodia |
| `src/coords.*`, `potential.*`, `montecarlo.*`, `integrator.h`, `vec3.h`, `units.h` | Orbit core, a byte-identical copy of nostos's ([PHY-012](docs/kb/issues/PHY-012-orbit-core-is-a-copy-of-nostos.md)); `integrator.h` is not used here |
| `scripts/build_boinc_all.sh` | CPU apps: 6 Linux and 3 Windows targets, static |
| `scripts/build_gpu_server.sh` | GPU apps: Linux CUDA, Linux OpenCL, Windows OpenCL |
| `scripts/build_gpu_windows.bat` | Windows CUDA app (MSVC and nvcc) |
| `scripts/embed_kernel.py` | Writes `src/phygas_kernel_cl.inc` |
| `scripts/prepare_clusters.py` | Hunt & Reffert 2024 `clusters.dat` to `young_clusters.txt` and `globular_clusters.txt` |
| `scripts/prepare_pulsars.py` | ATNF catalogue (HEASARC copy) to `pulsars.txt`; needs astropy |
| `scripts/make_star_catalog.py` | Gaia DR3 6-D CSV to the star target catalogue (`.txt.gz`); imports nostos code ([PHY-014](docs/kb/issues/PHY-014-make-star-catalog-needs-nostos-checkout.md)) |
| `docs/kb/` | Knowledge base (below) |

There is no vendored third-party code. `data/` and `build/` are in `.gitignore`.

## Invariants

Break one of these and results become silently wrong.

* **Output rows.** Two formats, each written in two places that must stay identical. Screen:
  `id cluster kind chi2 t d_nom_pc sig_pc p_r50 p_rtot vrel` (`main_phygas_boinc.cpp:165` GPU, `442` CPU). Pair:
  `id cluster kind n d05 d50 d95 t05 t50 t95 v05 v50 v95 f_r50 f_rtot` (`256` GPU, `453` CPU). The last line is
  `# done <nt>`. The server's validator and assimilator parse these. The CLIs print the same columns plus
  `age16 age84`.
* **Row order**: targets in the order of `in`, then clusters in file order. The GPU paths print in the same order.
* **Input formats**, one writer and one reader each: the cluster file (`prepare_clusters.py` to `ph_read_clusters`,
  19 fields), target lines (`prepare_pulsars.py`, `make_star_catalog.py` to `parse_target` in the app and
  `ph_read_targets` in the CLIs, 14 fields, `rv`/`e_rv` read as strings so `nan` works). A line that does not parse
  is skipped without a message ([PHY-007](docs/kb/issues/PHY-007-msvcrt-scanf-nan-drops-cluster-lines.md)).
* **Determinism.** A pair's result depends only on its seeds: `ph_seed(target, cluster)` (FNV-1a), and
  `ph_seed("lin"|"track", id)` per target and cluster. It does not depend on order, batch size or resume. Changing
  `ph_seed`, the RNG (splitmix64 and Box-Muller in `montecarlo.*`) or the order of draws changes every result.
* **CPU and GPU agree.** `phygas_kernel_body.h` is a literal port of `phygas.c`, `potential.c` and `montecarlo.c`,
  with constants copied (`0.5` Myr step at 193, `977.7922216807892` at 200) and the potential's parameters passed as
  9 doubles in the order of `main_phygas_boinc.cpp:79-80` and `190-191`. A change to the core needs the same change
  in the kernel body, then `python3 scripts/embed_kernel.py`. The authors measured byte-identical output on NVIDIA
  CUDA and OpenCL (commits 8bb46cb, b63a70a).
* **Floating point.** C core with `-std=c99` (GCC then does not fuse `a*b+c`), i686 and win32 with
  `-msse2 -mfpmath=sse`, `nvcc --fmad=false`, `#pragma OPENCL FP_CONTRACT OFF`. Do not add `-ffast-math`, do not
  switch to `-std=gnu99`, do not drop these flags. Identity across platforms is not established
  ([PHY-005](docs/kb/issues/PHY-005-results-may-differ-across-platforms-and-backends.md)).
* **Parameters**: `r_kernel` 0 is not passed through by the app ([PHY-008](docs/kb/issues/PHY-008-r-kernel-zero-cannot-be-selected.md)).

## Checkpoint/resume contract (what the code does at 0a3eda2)

* Files: `phygas_checkpoint` in the slot directory, one integer (the number of finished targets), text; `out`,
  appended.
* Authority: the checkpoint. With `start > 0`, `out` is opened with `"a"`, otherwise `"w"` (`main_phygas_boinc.cpp:413-421`).
  `out` is not cut back to the checkpoint, so rows after the last checkpoint are duplicated or torn
  ([PHY-001](docs/kb/issues/PHY-001-rows-after-last-checkpoint-duplicated-on-resume.md), reproduced).
* Writing: in place with `"w"`, unchecked, value not range-checked on reading
  ([PHY-002](docs/kb/issues/PHY-002-checkpoint-written-in-place-unchecked-no-range-check.md)); an open failure
  restarts from 0 ([PHY-015](docs/kb/issues/PHY-015-checkpoint-open-failure-restarts-from-zero.md)).
* Cadence: CPU path when `boinc_time_to_checkpoint()` returns 1 (the write is inside its critical section) and after
  the last target (459-464); GPU screen after every batch, GPU pair after every flush that completes targets, both
  without asking the timer ([PHY-003](docs/kb/issues/PHY-003-gpu-checkpoints-not-gated-by-time-to-checkpoint.md)).
  Critical sections: only the one `boinc_time_to_checkpoint()` opens; every 1 is paired (PAT-001 checked).
* Progress: `boinc_fraction_done()` after each target (CPU) or batch (GPU), not at resume; the first report comes
  within seconds anyway ([PHY-017](docs/kb/issues/PHY-017-progress-not-reported-at-resume.md), not a bug). Pulsar
  tasks are one target: no checkpoint,
  no progress until the end ([PHY-011](docs/kb/issues/PHY-011-pulsar-work-units-no-checkpoint-no-progress.md)).
* A resume must produce an `out` byte-identical to an uninterrupted run. The per-pair seeds make that possible;
  the trimming that would achieve it is missing (PHY-001).
* GPU errors end the task with `boinc_finish(1)`
  ([PHY-004](docs/kb/issues/PHY-004-gpu-errors-end-task-with-boinc-finish-1.md), question for the maintainers).

## Building

The scripts run on the bitboinc build server, from a directory that contains `src/` (the repository root).

* CPU, every Linux and Windows target: `PHYGAS_VERSION=<n> bash scripts/build_boinc_all.sh` (default version 1).
  Per target it runs `<cc> -O2 -std=c99 -I src -c src/<f>.c` for `phygas coords potential montecarlo`,
  `<cxx> -O2 -I src -I $B/api -I $B/lib -c src/main_phygas_boinc.cpp`, and links with
  `<cxx> -static ... $B/api/libboinc_api.a $B/lib/libboinc.a` (`-lpthread -lm` on Linux, `-lpsapi` on Windows).
  BOINC comes from `/opt/bitboinc-src/boinc-<target>`; compilers: distribution cross gcc for Linux,
  `/usr/bin/x86_64-w64-mingw32-*` and `i686-w64-mingw32-*`, llvm-mingw in `/opt/llvm-mingw/bin` for arm64.
  Output `out/phygas_v<n>_<os>_<arch>[.exe]`; the script prints the number of `GLIBC_` symbols (0 when static).
* GPU: `PHYGAS_VERSION=<n> bash scripts/build_gpu_server.sh` (default version 2; needs `sudo docker`). Linux CUDA in
  `nvidia/cuda:12.8.1-devel-ubuntu20.04` (`nvcc -O3 -std=c++17 --fmad=false`, sm_50 to sm_120, static cudart),
  Linux OpenCL in `ubuntu:20.04`, one binary per vendor (`ati` with `-DEXPECTED_GPU_VENDOR="amd"`, `nvidia`), and
  Windows OpenCL with the host's mingw-w64. BOINC is configured inside the containers from `/opt/bitboinc-src/boinc`
  (`--disable-client --disable-manager --disable-server --enable-apps`). The script prints the newest `GLIBC_`
  version each Linux binary needs.
* Windows CUDA: `scripts\build_gpu_windows.bat` on a Windows PC with Visual Studio 2022 Build Tools and CUDA 13.3
  (sm_75 to sm_120). The BOINC tree (built with `win_build`, x64 Release) is at a fixed path in the script: edit
  line 9 ([PHY-010](docs/kb/issues/PHY-010-windows-cuda-build-needs-developer-path.md)). Output
  `build\gpuwin\phygas_cuda_v<n>_windows_x86_64.exe`.
* macOS and Android (CPU) are built on their own hosts, "as for Nostos"; those scripts are not in this repository.
* After editing `src/phygas_kernel_body.h`: `python3 scripts/embed_kernel.py` and commit the regenerated `.inc`
  (at 0a3eda2 the committed file matches the generator's output).
* Run-time knobs of the GPU app: `PHYGAS_BATCH_STARS` (default 1024), `PHYGAS_BATCH_CLONES` (default 262144);
  `--device` is used only when `init_data.xml` gives no device; `--cl-platform` (default 0) is tried first when it
  matches the binary's vendor, otherwise the first matching platform is used.

## Tests

There are none in the repository. How to check a change today:

```
cc -O2 -std=c99 -I src src/phygas.c src/coords.c src/potential.c src/montecarlo.c src/phygas_screen.c -o phygas_screen -lm
cc -O2 -std=c99 -I src src/phygas.c src/coords.c src/potential.c src/montecarlo.c src/phygas_cli.c -o phygas_cli -lm
./phygas_screen clusters.txt stars.txt [chi2_max] [n_samp] [r_kernel] [control]
./phygas_cli clusters.txt targets.txt [n_clones] [dt_myr] [only=ID,...]
```

(both lines built and run with Apple clang on 2026-10-27). Run before and after the change on the same input and
`cmp` the outputs. The BOINC app needs BOINC's `api/` and `lib/`, or a small stand-in for the calls it uses
(`boinc_init_options`, `boinc_resolve_filename`, `boinc_fopen`, `boinc_get_init_data`, `boinc_time_to_checkpoint`,
`boinc_checkpoint_completed`, `boinc_fraction_done`, `boinc_finish`); kampylos's `tests/boinc_stub/` is a model.

* Core or kernel change: CLI `cmp`, and a GPU run compared with the CPU app on one work unit (needs a GPU host).
* Checkpoint change: quit and kill a run (a quit is `fflush(NULL)` then `exit`, not SIGKILL) and `cmp` with an
  uninterrupted run (PHY-001, Test).
* Compiler flags or math code: a cross-platform diff (PHY-005).

## Environment notes

* The build scripts depend on the build server's layout (above). The Windows CUDA script depends on one PC.
  Container images are pulled by tag and no build records the BOINC or compiler versions it used
  ([PHY-020](docs/kb/issues/PHY-020-build-dependencies-not-pinned-or-recorded.md)).
* GPU host code: every CUDA call and every OpenCL transfer and launch is checked; the OpenCL kernel-argument calls
  are not ([PHY-019](docs/kb/issues/PHY-019-opencl-kernel-arguments-set-unchecked.md)).
* Linux CPU binaries are fully static; the GPU builds target glibc 2.31 (Ubuntu 20.04; commit 8bb46cb says they
  need 2.29).
* Windows: text-mode output writes CRLF ([PHY-006](docs/kb/issues/PHY-006-output-and-checkpoint-text-mode-crlf-on-windows.md));
  a mingw-w64 build that uses `msvcrt.dll` may skip input lines with `nan` in a `%lf` column (PHY-007, to be
  confirmed). The Windows CUDA app has code only for
  compute capability 7.5 and newer ([PHY-018](docs/kb/issues/PHY-018-windows-cuda-build-only-sm75-and-newer.md)).
* OpenCL: the library is loaded at run time (no `-lOpenCL`); per-vendor binaries; fp64 is required and checked, so
  Intel GPUs and Apple Silicon cannot run the GPU app. `cl_dynload.c` is older than planodia's copy
  ([PHY-013](docs/kb/issues/PHY-013-cl-dynload-older-than-planodia-copy.md)).
* Apple clang 14 (checked on an M1) does not fuse `a*b+c` by default; newer clang does, so a Mac or llvm-mingw build
  may differ in the last bits (PHY-005).
* `scripts/build_boinc_all.sh`, `build_gpu_server.sh`, `build_gpu_windows.bat` and `embed_kernel.py` are near-copies
  of nostos's, with intended differences (names, source lists, the `PG_*` macros, Ubuntu 20.04 containers here and
  22.04 there, version defaults). They are not registered as shared code; compare them by hand when one changes.
* The comments of the orbit core and of `cl_dynload.h` refer to files that live in nostos (tests,
  `orbital_elements.c`, `NOTES_method_selection.md`) or nowhere ([PHY-009](docs/kb/issues/PHY-009-comments-name-missing-files-or-old-design.md)).

## Conventions for changes

* Small branches, one fix per commit with its test. The commit message names the issue (`KB: PHY-NNN`) and the
  pattern, if any (`Pattern: PAT-NNN`).
* Measured claims: say what was compared, on which platforms and inputs ("byte-identical on ..."). Date comments
  that record a finding.
* Keep the backends symmetric: the CPU loop, `gpu_screen` and `gpu_pair` print the same rows and should share one
  checkpoint helper; the kernel body follows the CPU code operation by operation.
* Never change science results silently: a change of seeds, draws, potential, integrator step or output precision
  is recorded with its measured effect and agreed with the maintainers.
* A change to a file listed as `shared-code` in `docs/kb/kb.conf` is copied to the other repositories in the same
  change (`python3 docs/kb/shared/kb.py status ../nostos ../planodia` shows drift).

## Not in this repository

* The server side: work generator (target lists, one pulsar per pulsar task, the sticky cluster file), validator,
  assimilator, plan classes. What the validator compares is not known here (PHY-005).
* The macOS and Android build scripts; the BOINC source (`/opt/bitboinc-src` on the build server).
* The input data: Gaia DR3 6-D CSV (downloaded by nostos's scripts), Hunt & Reffert 2024 from VizieR, the ATNF
  table from HEASARC.
* `nostos/scripts/make_work_units.py` (`zero_points`, used by `make_star_catalog.py`), the orbit core's tests
  (`nostos/tests/`) and `NOTES_method_selection.md`: in the nostos repository.
* Published results: the pulsar results page and the Zenodo records linked from README.md.

## Knowledge base

`docs/kb/` holds this repository's known bugs, platform traps and their fixes, and the bug patterns shared with the
other bitboinc applications. Start at [docs/kb/INDEX.md](docs/kb/INDEX.md): its first table, "Fix next", is the work
that is ready, with the next step for each issue. Before debugging a symptom, search it (`grep -ril "<words>" docs/kb`).
After a fix or a new finding, update the issue. Workflow: [docs/kb/shared/PROCESS.md](docs/kb/shared/PROCESS.md).

- Overview without opening files: `python3 docs/kb/shared/kb.py list --open [--platform windows] [--area checkpoint]`
- New report: `python3 docs/kb/shared/kb.py new issue "symptom in a few words"`
- A fix commit names its issue (`KB: PHY-NNN`) and, if any, its pattern (`Pattern: PAT-NNN`).
- Before a pull request: `python3 docs/kb/shared/kb.py index && python3 docs/kb/shared/kb.py lint --strict`
