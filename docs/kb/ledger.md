# Pattern ledger

One row per cross-repository pattern ([shared/patterns](shared/patterns)): has this repository been checked for it,
at which commit, and what was found. `kb.py sync` adds an `unchecked` row whenever a new pattern arrives; checking it
means following the pattern's "How to check a repository" section and filling in the row. Status values:
[shared/vocabulary.md](shared/vocabulary.md#ledger). A row that is `affected` or `fixed` names the local issue, and that
issue names the pattern in its `pattern:` field; its Evidence says where, in one clause, and the issue has the rest.
Write rows with `kb.py ledger` (date and commit filled in) rather than by hand.

When code that a row describes changes a lot, check the pattern again and update Checked, Commit and Evidence.

| Pattern | Status | Checked | Commit | Issue | Evidence |
|---|---|---|---|---|---|
| PAT-001 | not-affected | 2026-10-27 | 0a3eda2 |  | main_phygas_boinc.cpp:459-464: boinc_time_to_checkpoint() is evaluated once per target as the first operand of the condition and every 1 is followed by boinc_checkpoint_completed() on the only path (463); the GPU paths never call it (completed() alone, PHY-003, counter clamped at 0); no begin/end_critical_section calls |
| PAT-002 | affected | 2026-10-27 | 0a3eda2 | PHY-001 | main_phygas_boinc.cpp:413-421: start from phygas_checkpoint, out reopened with "a" and never cut back |
| PAT-003 | affected | 2026-10-27 | 0a3eda2 | PHY-002 | (a) "w" in place at main_phygas_boinc.cpp:169, 263, 461 |
| PAT-004 | affected | 2026-10-27 | 0a3eda2 | PHY-003 | GPU screen after every batch, GPU pair after each flush, never gated (main_phygas_boinc.cpp:168-172, 261-268) |
| PAT-005 | affected | 2026-10-27 | 0a3eda2 | PHY-015 | main_phygas_boinc.cpp:414-420: any failure to open phygas_checkpoint leaves start 0 and out is opened with "w" |
| PAT-006 | affected | 2026-10-27 | 0a3eda2 | PHY-016 | main_phygas_boinc.cpp:467-469: no ferror, fclose(out) unchecked, then boinc_finish(0) |
| PAT-007 | not-affected | 2026-10-27 | 0a3eda2 | PHY-017 | no boinc_fraction_done(start/nt) at resume (413-419), but the first report comes within seconds: cluster precomputation (about 0.1 s per 300 clusters on an M1) plus one target (465, about 0.02 s screen) or one GPU batch (172, 266; about 0.6 s on an RTX 5070 Ti per 8bb46cb); pulsar tasks have no resume (PHY-011) |
| PAT-008 | n/a | 2026-10-27 | 0a3eda2 |  | no quit_request, abort_request, boinc_get_status or direct_process_action in the repository; default BOINC_OPTIONS (main_phygas_boinc.cpp:337-342) |
| PAT-009 | n/a | 2026-10-27 | 0a3eda2 |  | no boinc_send_trickle_up; results go in the output file |
| PAT-010 | n/a | 2026-10-27 | 0a3eda2 |  | the worker only computes and reads regular files (fgets, fscanf); no sleep, poll, select or read calls in src; pg_now() reads CLOCK_MONOTONIC (phygas_gpu.h:51-55) |
| PAT-011 | not-affected | 2026-10-27 | 0a3eda2 |  | main_phygas_boinc.cpp:346-357: gpu_opencl_dev_index (OpenCL) or gpu_device_num (CUDA) when >= 0; --device (which the client sets to gpu_device_num) only as fallback; the code is under PG_GPU and PG_OPENCL, defined by every GPU build (build_gpu_server.sh:35, 48, 70; build_gpu_windows.bat:16) |
| PAT-012 | not-affected | 2026-10-27 | 0a3eda2 |  | phygas_opencl.c:39-58: platform chosen by the vendor of the per-vendor binary (EXPECTED_GPU_VENDOR amd or nvidia, defined in every OpenCL build: build_gpu_server.sh:51, 73) rather than aid.gpu_type; CL_DEVICE_TYPE_GPU for count and list; no CPU fallback; an index beyond the first matching platform's devices becomes 0 (56) |
| PAT-013 | affected | 2026-10-27 | 0a3eda2 | PHY-004 | fail() = boinc_finish(1) (main_phygas_boinc.cpp:23-27) for pg_init failure (357) and GPU runtime errors (425, 427) |
| PAT-014 | not-affected | 2026-10-27 | 0a3eda2 |  | phygas_opencl.c:62-67 reads CL_DEVICE_DOUBLE_FP_CONFIG and refuses a device without fp64 with a clear message; Intel GPUs not targeted (build_gpu_server.sh:8) |
| PAT-015 | not-affected | 2026-10-27 | 0a3eda2 |  | phygas_opencl.c:21, 27-29: local size 64 lowered to CL_KERNEL_WORK_GROUP_SIZE of every kernel (never above the device maximum); no __local memory; kernels check their index bound |
| PAT-016 | not-affected | 2026-10-27 | 0a3eda2 |  | launch sizes adapted to 0.25 s (phygas_gpu.h:33, 57-65; main_phygas_boinc.cpp:113-127, 150-156, 225-231); in pair mode one flush can hold up to 4x the clones of the flush the slice was sized on (cap = 4 x batch, 196), so about 1 s in the worst case; not measured |
| PAT-017 | n/a | 2026-10-27 | 0a3eda2 |  | no Android OpenCL build in this repository (build_gpu_server.sh: Linux CUDA/OpenCL, Windows OpenCL; build_gpu_windows.bat: Windows CUDA; Android CPU is built elsewhere); cl_dynload.c lacks planodia 3f6c1b7, see PHY-013 |
| PAT-018 | affected | 2026-10-27 | 0a3eda2 | PHY-005 | not measured across platforms; the winarm64 llvm-mingw build may contract (build_boinc_all.sh:43) |
| PAT-019 | affected | 2026-10-27 | 0a3eda2 | PHY-006 | out and phygas_checkpoint opened with "w"/"a" (main_phygas_boinc.cpp:169, 263, 420, 461) in the mingw-w64 and MSVC Windows builds |
| PAT-020 | n/a | 2026-10-27 | 0a3eda2 |  | no threads, thread_local, OpenMP or std::async in src |
| PAT-021 | affected | 2026-10-27 | 0a3eda2 | PHY-010 | build_gpu_windows.bat:9 sets BOINC to a directory in one user's profile, not overridable |
| PAT-022 | not-affected | 2026-10-27 | 0a3eda2 |  | CPU Linux builds are -static and their GLIBC_ symbols are counted (build_boinc_all.sh:26, 29); GPU builds in Ubuntu 20.04 containers (build_gpu_server.sh:59, 61), the newest GLIBC_ printed at the end (78-80) but not enforced; commit 8bb46cb: GPU builds need GLIBC 2.29 |
| PAT-023 | n/a | 2026-10-27 | 0a3eda2 |  | no downloads and no resumable progress record: prepare_clusters.py, prepare_pulsars.py and make_star_catalog.py read a local file in one pass (the last writes <out>.part and renames it, 37, 71); the Gaia download is nostos's |
| PAT-024 | affected | 2026-10-27 | 0a3eda2 | PHY-018 | Linux CUDA 12.8.1 builds sm_50 to sm_120 (build_gpu_server.sh:31), Windows CUDA 13.3 only sm_75 to sm_120 (build_gpu_windows.bat:5, 10), both with compute_120 PTX |
| PAT-025 | n/a | 2026-10-27 | 0a3eda2 |  | the BOINC app starts no threads (no std::thread, pthread_create, OpenMP, std::async or CreateThread in src) |
| PAT-026 | not-affected | 2026-10-27 | 0a3eda2 |  | only sort reaching the output: qsort of clone distances, times and speeds before the quantiles (phygas.c:230-232); ties are equal doubles, so their order cannot change a quantile; values are finite and never -0.0; no sorting in the scripts |
| PAT-027 | affected | 2026-10-27 | 0a3eda2 | PHY-019 | OpenCL ARG_B/ARG_D/ARG_I ignore the clSetKernelArg result (phygas_opencl.c:117-119) |
| PAT-028 | not-affected | 2026-10-27 | 0a3eda2 |  | guards PG_GPU, PG_OPENCL (main_phygas_boinc.cpp:29, 339, 345, 350, 423) and EXPECTED_GPU_VENDOR (phygas_opencl.c:12) are defined by every shipped GPU build (build_gpu_server.sh:35, 48, 51, 70, 73; build_gpu_windows.bat:16); a missing one would be silent (CPU app, or no vendor filter) |
| PAT-029 | affected | 2026-10-27 | 0a3eda2 | PHY-020 | build_gpu_server.sh:59, 61 pull images by tag, :18 apt-get install without versions |
| PAT-030 | n/a | 2026-10-27 | 0a3eda2 |  | no boinc_temporary_exit() and no deliberate exit without boinc_finish() in src/ (git grep) |
