# Unlock Procedure, Safety Checks, and Rollback

## Operational Mechanism

The speed selectors controlling Turing FP16 tensor core rate reside in protected Falcon FECS registers (`0x409664` for SS0 and `0x40966c` for SS1), gated by privilege mask register `0x409650` (PLM). Under factory configuration, writing to these registers from host software has no effect because write permissions are restricted to security level 3 (heavy-secure mode).

The unlock operates by running a signed Booter sequence with an adapted payload (the V67 payload) inside SEC2 before the official NVIDIA driver initializes the card. The payload adjusts the privilege mask, programs the speed selectors to full rate (SS0 = `0x88888888`, SS1 = `0x8`), releases ACR mutex 3, and finalizes through standard firmware paths.

Because official driver initialization configures Write-Protected Region 2 (WPR2) and transitions the security processor into a warm state, this sequence must execute in a clean pre-driver environment. Attempting to run the module after a driver has bound will result in `-EBUSY` or `-EALREADY`.

## Exact Step-by-Step Procedure

The following procedure was executed on our test system:

### 1. Build and Firmware Verification

1. Clone or pull the repository at commit `1d379b7`.
2. Generate derived firmware assets:
   ```sh
   make assets
   ```
3. Verify generated asset hashes against `firmware/artifacts-610.43.03.json`.
4. Build the kernel module against the target kernel header tree:
   ```sh
   make -C kernel/turing_unlock KDIR=/lib/modules/$(uname -r)/build
   ```
5. Install firmware files to the system firmware directory:
   ```sh
   sudo make install-firmware
   ```
   Confirm that all 6 auxiliary firmware binaries are in `/usr/lib/firmware/turing-unlock/610.43.03/`.

### 2. Driver Suppression and Clean Boot

1. Stop all user processes and background services utilizing the GPUs (such as AI model inference runtimes or display managers).
2. Configure modprobe rules to suppress automatic NVIDIA driver loading on the subsequent boot (for example, setting `/bin/false` install overrides or blacklisting `nvidia`, `nvidia_modeset`, `nvidia_drm`, and `nvidia_uvm`).
3. Reboot the machine into the driver-suppressed environment.
4. Verify that no GPU driver modules are loaded (`lsmod` contains no NVIDIA or Nouveau modules) and confirm that PCI devices `0000:01:00.0` and `0000:02:00.0` have no driver link in `/sys/bus/pci/devices/<bdf>/driver`.

### 3. Canary Unlock on Secondary GPU

To minimize operational risk on the display-capable bus, unlock the secondary non-display GPU first:

1. Insert the module targeting the secondary device:
   ```sh
   sudo insmod kernel/turing_unlock/turing_unlock.ko targets=0000:02:00.0
   ```
2. Check the kernel log buffer (`dmesg`):
   - Confirm readback line: `unlock result: ss0=0x88888888 ss1=0x8 readout=0x0`
   - Confirm status line: `TU102 FEAT UNLOCKED`
   - Confirm PCI line: `pci 0000:02:00.0: turing_unlock: unlocked`
3. Verify that no DMA retention warning appears.

### 4. Volatility and Reset Verification

1. Reboot the system while maintaining driver suppression.
2. Verify that upon reboot, the GPU resets its internal registers back to factory locked values. This confirms that the modification is strictly volatile and leaves no permanent flash changes.

### 5. Dual-Card Simultaneous Unlock

1. In the driver-suppressed boot, invoke the module with comma-separated targets for both GPUs:
   ```sh
   sudo insmod kernel/turing_unlock/turing_unlock.ko targets=0000:01:00.0,0000:02:00.0
   ```
2. Check the kernel log to confirm that each GPU logs its respective `TU102 FEAT UNLOCKED` and `pci <bdf>: turing_unlock: unlocked` entries with zero errors.

### 6. Driver Handoff

1. Remove the temporary driver suppression modprobe rules.
2. Load the standard NVIDIA open driver modules:
   ```sh
   sudo modprobe nvidia
   sudo modprobe nvidia_uvm
   ```
3. Inspect system status via `nvidia-smi`:
   - Both cards enumerate normally.
   - Addressable VRAM confirms 22,528 MiB per card.
   - NVLink status reports both physical links active at 25.781 GB/s and topology reports NV2.

## Safety Checks and Rollback Strategy

- **Volatile Execution (Safe Rollback)**:
  The unlock sequence modifies volatile SRAM and register state within the GPU. It does not flash the VBIOS chip, modify EEPROM contents, or blow hardware fuses. Consequently, performing a standard reboot or power-cycle immediately clears all overrides and returns the hardware to its stock factory state.
- **DMA Quiescence Protection**:
  If the security processor fails to halt within the polling window, the kernel module will refuse to complete cleanup and will log `retaining DMA resources until power off`. In such an error condition, the module deliberately retains its coherent DMA allocations to prevent host RAM corruption from dangling device transfers. If this occurs:
  - Do NOT attempt to force-load the NVIDIA driver.
  - Do NOT reload the unlock module.
  - Execute a clean system power cycle (full power off) to reset hardware state.

## Power and Thermal Caution

- **High Power Demand**:
  Full-rate FP16 tensor core operation doubles the instruction issue rate for FP16 matrix operations with FP32 accumulation. This significantly increases instantaneous and sustained power draw across the GPU core and voltage regulator modules (VRMs).
- **22GB Hardware Mods**:
  Cards modified with 22GB memory configurations utilize higher-density memory chips or modified power supply circuitry. Increased tensor core compute density places additional thermal stress on these components.
- **Power Limiting**:
  All benchmarks and validation runs were executed with a strict 250W power cap enforced via `nvidia-smi -pl 250`. Users are strongly advised to enforce explicit power caps and verify robust chassis airflow before deploying sustained unlocked workloads. Under identical 250W caps, unlocked full-load thermals measured approximately 1 to 2 degrees Celsius higher than locked baseline operation.
