---
id: PHY-019
title: OpenCL kernel arguments are set without checking clSetKernelArg, so a failure can reuse the previous launch's value
type: bug
status: confirmed
severity: low
platforms: [opencl]
area: gpu-runtime
next: In phygas_opencl.c, make ARG_B/ARG_D/ARG_I return the clSetKernelArg error and stop the pg_* call with a message when one fails
pattern: PAT-027
aliases: []
reported: 2026-10-27
source: pattern ledger check (PAT-027), 2026-10-27
where: src/phygas_opencl.c:117-119 @ 0a3eda2
fixed_in:
---

## Symptom

None reported. If a `clSetKernelArg` call failed after an earlier launch had set that argument, the kernel would run
with the old value (for example the previous slice's start `k0`, `p0` or `h0`) and the host would read back
plausible but wrong numbers, written to `out` as a normal result.

## Cause

Verified by reading `src/phygas_opencl.c` at 0a3eda2. The macros `ARG_B`, `ARG_D` and `ARG_I` (117-119) call
`cl_api.SetKernelArg` and ignore its return value; every launch sets its arguments through them (`pg_lin` 130-137,
`pg_chi2` 139-146, `pg_finish` 148-156, `pg_pair` 158-163). Everything else on the output path is checked: buffer
writes and reads (`pg_put`, `pg_get`, 102-113), `clEnqueueNDRangeKernel` and `clFinish` (`run`, 121-128). The CUDA
backend checks every copy and launch (`CK` around `cudaMemcpy`, `cudaGetLastError`, `cudaDeviceSynchronize`,
`src/phygas_cuda.cu:16-17, 40-76`). Argument sizes and indices match the kernel signatures in
`src/phygas_kernel_body.h`, so a failure would need an unusual driver error (out of host memory or resources); not
seen.

## Fix

Not fixed. Make the macros return the error and have each `pg_*` function stop with an error message when any
argument call fails, as `run()` does for the launch; the caller then ends the task (PHY-004 decides how).

## Test

A test build whose `SetKernelArg` wrapper fails on, say, the 100th call must end with an error, not with output.

## Log

- 2026-10-27: found while checking PAT-027 at 0a3eda2.
