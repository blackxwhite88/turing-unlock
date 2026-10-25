# Provenance, Credits, and Research Background

## Upstream Lineage and Credits

The work presented here builds upon contributions across the GPU research community. Full attribution and thanks belong to the original authors:

- **nlzy/turing-unlock** (commit `1d379b7`)
  - Repository: https://github.com/nlzy/turing-unlock
  - Role: The direct upstream base for this fork. It provides a Linux kernel module that parses native VBIOS FWSEC dynamically at boot time, prepares firmware assets, loads Booter and the V67 payload on GSP and SEC2, applies the speed selector values (`0x88888888` and `0x8`), releases ACR mutex 3, and completes clean handoff to the OS.

- **Zagitta/gpu-unlock** (commit `a9b0c75`)
  - Repository: https://github.com/Zagitta/gpu-unlock
  - Support the author: https://buymeacoffee.com/zagitta
  - Role: The foundational Ampere (GA10x) kernel module unlocker that `turing-unlock` adapts. Pioneered the combination of the V67/PLM technique with pre-driver kernel module timing to override hardware speed selectors before official driver initialization.

- **KohakuBlueleaf/KohakUnlock** (commit `eda111e`, GPL-2.0)
  - Repository: https://github.com/KohakuBlueleaf/KohakUnlock
  - Role: Early Turing Linux module research. Explored pre-driver Linux execution and noncontiguous DMA buffer handling on Turing TU102 hardware.

- **xrip/cmp50hx-unlock**
  - Repository: https://github.com/xrip/cmp50hx-unlock
  - Role: Foundational reverse-engineering on CMP 50HX and Turing TU102 architecture. Identified the protected FECS override registers (`0x409650`, `0x409664`, `0x40966c`) and established the V67 payload structure.

- **haohervchb/Turing-unlock** (commit `a188b24`)
  - Repository: https://github.com/haohervchb/Turing-unlock
  - Role: UEFI-level pre-OS unlock research on an ASUS RTX 2080 Ti card. Demonstrated volatile pre-boot register overrides and provided independent confirmation of 2x FP16 with FP32-accumulation compute scaling.

- **weicj/vLLM-2080Ti-Definitive**
  - Repository: https://github.com/weicj/vLLM-2080Ti-Definitive
  - Role: Specialized vLLM inference engine optimizations for dual RTX 2080 Ti 22GB configurations. Provided the operational reference and baseline profiling configuration used in our real-world serving benchmarks.

## Code and License Status

- **Unmodified Upstream Code**: The functional code, kernel module implementation, firmware generation tools, and scripts in this repository are completely unmodified from upstream `nlzy/turing-unlock` at commit `1d379b7`.
- **License Retention**: The upstream repository contains no standalone top-level license file. All licensing terms are defined by the files in `licenses/` (GPL-2.0 for the kernel module and packaging scripts, and NVIDIA MIT notices for driver interface components). All rights remain with their respective original authors.

## What We Tried First and Why We Dropped It

Before adopting `turing-unlock`, an earlier investigation explored an in-house custom loader design paired with a staged firmware security planner to manually stage Booter and execution images. That custom loader path was abandoned once public work demonstrated that a clean, native Linux kernel module could dynamically extract the card's native FWSEC from VBIOS at boot, execute the necessary security processor handshake, and cleanly hand control over to the standard NVIDIA Linux driver without requiring custom host firmware injection or manual baremetal staging.
