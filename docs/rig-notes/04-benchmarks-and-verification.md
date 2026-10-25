# Benchmark Results and Verification Matrix

## Benchmark Methodology and Numbers

Performance evaluation was conducted across two distinct testing tiers: isolated synthetic matrix multiplication (testing peak compute throughput) and real-world large language model serving (testing practical end-to-end impact).

### 1. Synthetic FP16 Matrix Multiplication (GEMM)

- **Test Description**: Square dense matrix multiplication ($N \times N$) in FP16 precision with FP32 accumulation using PyTorch, measured over sustained multi-second runs.
- **Power Envelope**: Strict 250W power limit enforced per card.

| Matrix Dimension ($N$) | Locked Baseline (GPU0) | Unlocked (GPU0) | Unlocked (GPU1) | Relative Increase |
|:---|---:|---:|---:|---:|
| 1024 | 46 TFLOPS | 59 TFLOPS | 60 TFLOPS | +28% to +30% |
| 2048 | 50 TFLOPS | 72 TFLOPS | 73 TFLOPS | +44% to +46% |
| 4096 | 50 TFLOPS | 72 TFLOPS | 74 TFLOPS | +44% to +48% |
| 8192 | 51 TFLOPS | 72 TFLOPS | 72 TFLOPS | +41% |

When running matrix multiplications on both cards simultaneously under the same 250W power cap, each card sustained 68 to 71 TFLOPS.

#### Architectural Mechanism Behind the Numbers

On the NVIDIA Turing (SM75) architecture:
- FP16 tensor operations with FP16 accumulation execute at full hardware rate (1024 FLOP/SM/clock, reaching approximately 116 to 125 TFLOPS on TU102).
- On consumer GeForce parts (such as the RTX 2080 Ti), FP16 tensor operations with FP32 accumulation are intentionally throttled to half rate (512 FLOP/SM/clock, capping throughput at approximately 45 to 51 TFLOPS).
- On professional Turing parts (such as Quadro RTX 6000 and Tesla T4), FP16 tensor operations with FP32 accumulation execute at full rate (1024 FLOP/SM/clock).
- Overriding the speed selectors lifts this artificial instruction issue cap, allowing FP16 with FP32 accumulation to reach the full hardware throughput of the SMs.

### 2. Real-World LLM Serving Workload

- **Configuration**: Serving a 27B parameter model (Qwen 27B) using vLLM in a Tensor Parallelism 2 (TP=2) configuration spanned across both cards over the NVLink bridge.
- **Workload Profile**: Single request consisting of approximately 126k prompt context and 2k output length, operating under a 250W power limit per card.
- **Results**:
  - **Prefill Throughput (Compute-Bound)**:
    - Locked Baseline: 804 to 824 tok/s
    - Unlocked: 919 to 942 tok/s (approximately +14% improvement)
  - **Decode Throughput (Memory Bandwidth-Bound)**:
    - Locked Baseline: 158 to 163 tok/s
    - Unlocked: 168 to 169 tok/s (within standard run variance; decode is memory-bound rather than compute-bound)
  - **Thermals**: Operating under the identical 250W power limit, peak operating temperatures rose by approximately 1 to 2 degrees Celsius (peak 75 C on GPU0 and 73 C on GPU1).

## Verification Matrix

To maintain rigorous transparency, the table below documents what has been experimentally verified on this hardware setup versus what remains unverified.

| Area | Status | Evidence and Findings |
|:---|:---|:---|
| **Dual 2080 Ti 22GB Cards** | Verified | Validated on dual TU102 cards (`10de:1e04`, subsystem `10de:12ae`, VBIOS `90.02.17.00.64`). |
| **Open Driver 595.91.07** | Verified | Successfully bound and operated with Linux kernel `7.0.0-38-generic`. |
| **Usable VRAM Post-Unlock** | Verified | Both cards report full 22,528 MiB addressable memory with zero allocation errors. |
| **NVLink Interconnect State** | Verified | Topology recognized as NV2; 2 links active at 25.781 GB/s per card. |
| **Compute Scaling** | Verified | FP16 GEMM compute and LLM prefill speedups measured and confirmed. |
| **Rollback via Reboot** | Verified | System reboot returns GPU registers to stock locked configuration. |
| **Persistence Across Reboot** | **NOT Verified** | Hardware register overrides are volatile. Automated reapplication at early boot is **NOT yet verified** on this rig. |
| **NVLink Saturated Bandwidth** | Unknown / Not Verified | Link state is verified, but sustained bidirectional saturation benchmarking was not run. |
| **Proprietary Driver 595.x** | Unknown / Not Verified | Only open kernel modules (`595.91.07`) were tested on this setup. Upstream tested proprietary `575.51.03` on `1e07`. |
| **Other Card / VBIOS Models** | Unknown / Not Verified | Other vendor boards, stock 11GB boards, or different VBIOS versions remain untested here. |
| **Partial Failure Recovery** | Unknown / Not Verified | Dual-card unlock succeeded on both targets simultaneously; asymmetric failure behavior was not observed. |
