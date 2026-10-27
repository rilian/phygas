---
id: PHY-018
title: The Windows CUDA build has device code only for compute capability 7.5 and newer
type: question
status: new
severity: medium
platforms: [windows, cuda]
area: build
next: Decide: does the Windows CUDA plan class require compute capability 7.5+ and a CUDA 13.3 driver? If not, restrict it or build with CUDA 12.x
pattern: PAT-024
aliases: []
reported: 2026-10-27
source: reading of the build scripts, 2026-10-27
where: scripts/build_gpu_windows.bat:10 @ 0a3eda2
fixed_in:
---

## Symptom

None reported. Expected on a Windows host with a Maxwell, Pascal or Volta GPU (GTX 9xx, GTX 10xx, Titan V) if the
server sends it the Windows CUDA app: the first kernel launch fails ("no kernel image is available for execution on
the device"), and the task ends as a computation error through `fail()` (PHY-004).

## Cause

Read at 0a3eda2. `scripts/build_gpu_windows.bat:10` builds for sm_75, 80, 86, 89, 90, 100 and 120 plus compute_120
PTX, with CUDA 13.3 (5); CUDA 13 no longer compiles for compute capability below 7.5. The Linux CUDA build
(`scripts/build_gpu_server.sh:31`, CUDA 12.8.1) also covers sm_50, 60 and 70. PTX for compute_120 cannot be compiled
for older GPUs at run time. nvcc links the CUDA runtime statically by default, so the host's driver must also be
new enough for CUDA 13.3.

Both CUDA builds embed compute_120 PTX for newer GPUs. Nothing in the repository records the minimum compute
capability of either build, and the app does not print it.

## Fix

Question for the maintainers: does the Windows CUDA plan class require compute capability 7.5 or newer and a driver
that supports CUDA 13.3? The server configuration is not in this repository. Hosts with older NVIDIA GPUs can run
the OpenCL NVIDIA app (fp64 is checked there). If the plan class does not restrict it, either restrict it or build
the Windows CUDA app with CUDA 12.x and the same gencodes as Linux.

## Test

On a Windows host with a GTX 10xx GPU: the CUDA app must not be sent, or must run.

## Log

- 2026-10-27: noticed while reading the build scripts at 0a3eda2; not run on hardware.
- 2026-10-27: linked to PAT-024 (GPU binaries compiled for fewer architectures than the plan class allows), which
  was written from this class of finding; status stays new until the plan class is known.
