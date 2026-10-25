# Operational Pitfalls and Engineering Notes

This document collects operational pitfalls, driver quirks, and design observations gathered during testing.

## Pitfalls Encountered

### 1. Pre-Driver Timing and Warm State Rejection

- **Symptom**: Invoking `turing_unlock.ko` fails with `-EBUSY` or `-EALREADY`.
- **Cause**: The official NVIDIA driver initializes Write-Protected Region 2 (WPR2) and transitions the GPU security processor into a warm state. Once this initialization has taken place, the Booter and V67 payload cannot execute.
- **Remedy**: The card must be in a cold, unbound state. The driver must be suppressed (via modprobe install rules or blacklisting) prior to boot so that the system enters early userspace with no GPU drivers bound to the PCI functions.

### 2. Module Return Code Ambiguity

- **Symptom**: `insmod` or `modprobe` exits with return code `0`, but the GPU remains locked.
- **Cause**: The kernel module's `init_module` entrypoint returns `0` upon completing its iteration across targets, even if a target card failed its unlock transaction.
- **Remedy**: Always inspect the kernel log buffer (`dmesg`) for explicit confirmation lines:
  - `unlock result: ss0=0x88888888 ss1=0x8 readout=0x0`
  - `TU102 FEAT UNLOCKED`
  - `pci <bdf>: turing_unlock: unlocked`
  Do not treat a zero exit code from `insmod` as verification of an unlocked card.

### 3. Benign Display Engine Assertion on Headless Cards

- **Symptom**: The NVIDIA open kernel driver logs `kern_disp_0400.c:223` during driver binding.
- **Cause**: In headless configurations where no physical monitor cables are connected to the card, the open driver's display management subsystem asserts on display engine state during device probe.
- **Remedy**: Testing verified that this assertion is harmless. All compute, CUDA, NVLink, and 22 GB memory management operations function normally.

### 4. Console Framebuffer Detachment During Function-Level Reset

- **Symptom**: The system console display goes black during module insertion and stays blank until a graphics driver binds.
- **Cause**: The unlock sequence performs a PCIe Function-Level Reset (`pcie_flr`) and calls `aperture_remove_conflicting_pci_devices` to clear stale memory decodes. If a card is driving an EFI or VESA console framebuffer, the framebuffer is detached by the kernel aperture cleanup.
- **Remedy**: Expected behavior on cards bound to primary display apertures. Running the system headlessly or relying on serial or network management avoids unexpected display drops.

### 5. DMA Buffer Retention on Firmware Halt Failure

- **Symptom**: Kernel logs indicate `firmware did not stop; retaining DMA resources until power off`.
- **Cause**: If GSP or SEC2 fails to halt cleanly within the 5-second polling timeout, the module deliberately keeps coherent DMA allocations mapped and increments its own module reference count. This fail-safe prevents host physical RAM from being overwritten by un-quiesced GPU DMA engines.
- **Remedy**: In this failure state, never force driver binding or module unloading. The only supported recovery path is a full system power cycle (power off, then power on).

### 6. Volatility Across Reboots

- **Symptom**: After a system reboot, FP16 matrix compute rates return to locked baseline numbers.
- **Cause**: Overriding speed selectors modifies volatile register and SRAM state inside the GPU; no permanent firmware or flash modifications are made.
- **Status**: The unlock does not survive a reboot. Automated persistent reapplication (such as modprobe install hooks or initramfs integration) has not been verified on this setup across reboots.
