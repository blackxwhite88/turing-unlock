# Dual RTX 2080 Ti 22GB Unlock and Verification Notes

This directory documents the setup, execution, and verification of the NVIDIA Turing TU102 tensor core rate unlock on a dual RTX 2080 Ti 22GB system connected via NVLink.

The underlying codebase of this fork is completely unmodified from upstream `nlzy/turing-unlock` at commit `1d379b7`. The upstream repository provides no standalone root license file; licensing is governed by the files in `licenses/` (GPL-2.0 for module and packaging code, and NVIDIA MIT notices for driver headers). All rights remain with the original authors.

## Documentation Index

1. [Provenance, Credits, and Research Background](01-provenance-and-credits.md)
   Details upstream projects, architecture discovery credits, repository lineage, licensing status, and why earlier custom-loader approaches were discontinued.

2. [Tested Hardware and Environment](02-hardware-and-environment.md)
   Covers board specifications (2x RTX 2080 Ti 22GB modded cards, device ID `10de:1e04`, subsystem `10de:12ae`), Linux kernel, open driver version, and NVLink topology.

3. [Unlock Procedure, Safety Checks, and Rollback](03-procedure-and-safety.md)
   Provides the exact step-by-step procedure used to unlock the cards, pre-driver suppression requirements, safety checks, DMA retention guards, rollback instructions, and power cautions.

4. [Benchmark Results and Verification Matrix](04-benchmarks-and-verification.md)
   Presents before and after performance benchmarks (synthetic PyTorch FP16 matrix multiplication and real-world vLLM serving throughput), alongside an explicit breakdown of what has been verified versus what remains unverified.

5. [Operational Pitfalls and Engineering Notes](05-pitfalls-and-notes.md)
   Documents real-world pitfalls encountered during testing, including module return code quirks, display driver assertions, console detachment, and hardware volatility.
