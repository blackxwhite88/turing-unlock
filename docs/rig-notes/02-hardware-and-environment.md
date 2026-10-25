# Tested Hardware and Environment

## Tested Hardware Configuration

Testing and verification were conducted on a dual-GPU workstation with modified RTX 2080 Ti cards:

- **GPU Model**: 2x NVIDIA GeForce RTX 2080 Ti (TU102 die).
- **Memory Capacity**: 22 GB GDDR6 per card (physical memory upgrade from stock 11 GB to 22 GB, reporting 22,528 MiB addressable memory).
- **PCI Device Identification**:
  - Vendor and Device ID: `10de:1e04` (the 1e04 hardware variant; upstream README tested Dell reference `10de:1e07`).
  - Subsystem Vendor and Device ID: `10de:12ae`.
- **VBIOS Firmware Version**: `90.02.17.00.64` on both cards.
- **Bus Topology**:
  - Primary Card (Boot display card): PCI slot `0000:01:00.0` (`boot_vga = 1`).
  - Secondary Card: PCI slot `0000:02:00.0` (`boot_vga = 0`).
  - Display attachment: Headless configuration (no physical monitor cables connected to either card).
- **GPU Interconnect**:
  - NVLink bridge installed across both cards.
  - Topology detected as NVLink Generation 2 (NV2).
  - Link speed: 2 physical links per card operating at 25.781 GB/s per link.

## Operating System and Platform

- **Distribution**: Ubuntu 26.04 LTS.
- **Linux Kernel**: `7.0.0-38-generic` (x86_64 architecture).
- **Compiler Suite**: GCC 15.2.0.
- **Firmware Security**: UEFI Secure Boot disabled.

## NVIDIA Driver

- **Driver Version**: `595.91.07`.
- **Driver Type**: NVIDIA Open GPU Kernel Modules (`nvidia.ko`, `nvidia-modeset.ko`, `nvidia-drm.ko`, `nvidia-uvm.ko`).
- **Compatibility Note**: Upstream documentation noted verification on driver `575.51.03` with a Dell `10de:1e07` card. Our validation confirms that the open driver line at version `595.91.07` binds and operates properly on dual `10de:1e04` cards post-unlock.
