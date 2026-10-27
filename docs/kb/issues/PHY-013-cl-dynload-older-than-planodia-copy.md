---
id: PHY-013
title: cl_dynload.c is an older copy without planodia's Android linker-namespace fallback
type: bug
status: confirmed
severity: info
platforms: [android, opencl]
area: device-selection
next: Copy planodia's src/cl_dynload.c (3f6c1b7) here and to nostos, so that the three copies are identical
pattern:
aliases: []
reported: 2026-10-27
source: comparison of the shared copies, 2026-10-27
where: src/cl_dynload.c:26-64 @ 0a3eda2
fixed_in:
---

## Symptom

None today. An Android OpenCL build made from this copy would find no OpenCL library on phones whose vendor does
not list `libOpenCL.so` as public (PAT-017).

## Cause

Compared 2026-10-27: `src/cl_dynload.c` is identical to nostos's copy (nostos eb7029e) and differs from planodia's
(planodia 3f6c1b7, "Android: open the OpenCL library through the vendor linker namespaces when a plain dlopen
fails", 2026-10-12). The phygas copy lacks the `__ANDROID__` parts: `android_dlopen_ext()` through the `sphal`,
`vndk` and `default` namespaces, the vendor paths, a private namespace as the last resort, and `dlerror()` in the
failure message. `cl_dynload.h` is identical in all three.

phygas has no Android OpenCL build: `scripts/build_gpu_server.sh` builds Linux CUDA, Linux OpenCL and Windows
OpenCL, `scripts/build_gpu_windows.bat` Windows CUDA; the Android build made elsewhere is the CPU app
(`scripts/build_boinc_all.sh:4-5`). So the ledger row for PAT-017 is `n/a` and this drift has no effect now.

## Fix

Not done. Copy planodia's `src/cl_dynload.c` here and to nostos, so that the three copies are identical again
(`kb.conf` registers them; `kb.py status ../nostos ../planodia` shows the drift). Required before an Android GPU
build is added.

## Test

`cmp src/cl_dynload.c ../planodia/src/cl_dynload.c`; the Linux and Windows OpenCL builds still compile (the new code
is inside `#ifdef __ANDROID__`).

## Log

- 2026-10-27: found by comparing the registered shared copies with nostos eb7029e and planodia 3f6c1b7.
