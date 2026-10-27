---
id: PHY-020
title: GPU build containers and packages are not pinned, and no build records the BOINC or compiler versions it used
type: bug
status: confirmed
severity: low
platforms: [all]
area: build
next: Pull the build images by digest; print gcc, nvcc, mingw-w64 and llvm-mingw versions and the BOINC commit into the build log
pattern: PAT-029
aliases: []
reported: 2026-10-27
source: pattern ledger check (PAT-029), 2026-10-27
where: scripts/build_gpu_server.sh:18 @ 0a3eda2
fixed_in:
---

## Symptom

None reported. Two builds of the same commit can differ, and an old release cannot be rebuilt exactly: for example to
find out whether a result difference comes from the compiler (FMA contraction, PHY-005) or the C runtime (PHY-007).

## Cause

Read at 0a3eda2.

* `scripts/build_gpu_server.sh:59, 61` pull `nvidia/cuda:12.8.1-devel-ubuntu20.04` and `ubuntu:20.04` by tag, not by
  digest; the publishers rebuild images under the same tag.
* `build_gpu_server.sh:18` installs `build-essential m4 pkg-config file` (and `opencl-headers`) with `apt-get`
  without versions. The compiler and the static parts of libstdc++ and libgcc
  (`-static-libstdc++ -static-libgcc`) end up in the Linux GPU binaries.
* BOINC is not fetched but taken from the build machine (`/opt/bitboinc-src/boinc*`, and a developer's directory for
  Windows CUDA, PHY-010), and the cross compilers and llvm-mingw too (`scripts/build_boinc_all.sh:14, 35-43`). No
  script prints which BOINC commit or compiler versions were used. Only CUDA is versioned by path or image tag
  (12.8.1 on Linux, 13.3 on Windows).

## Fix

Not fixed. Pull the images by digest (or a dated tag) and note it in the script; print `gcc --version`,
`nvcc --version`, the llvm-mingw and mingw-w64 versions and `git -C <boinc dir> rev-parse HEAD` into the build log
and keep that log with the release. Pinning apt package versions on Ubuntu 20.04 is optional once the image digest is
fixed.

## Test

Build the same commit twice in clean containers a while apart and compare the recorded versions and the output of a
small work unit.

## Log

- 2026-10-27: found while checking PAT-029 at 0a3eda2.
