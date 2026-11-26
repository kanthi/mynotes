---
title: Nsight Compute Basics for Kernel Authors
---

# Nsight Compute Basics for Kernel Authors

**NVIDIA Nsight Compute (`ncu`)** is an interactive hardware-counter profiler that replays individual GPU kernels to tell you *why* a kernel is slow — down to the exact warp stall reason, shared-memory bank conflict count, and HBM vs. Tensor Core utilization. Instead of guessing which optimization to try next, you read one `ncu` report and let the bottleneck dictate the edit.

## The four-step Nsight Compute reading workflow

1. **GPU Speed of Light (SOL)**: Compare `Compute (SM) Throughput [%]` vs. `Memory Throughput [%]`.
   - Memory > 70%, Compute < 40% → **Memory-bound** (fix coalescing, fuse ops, increase tiling/reuse, quantize weights).
   - Compute > 70% → **Compute-bound** (move to Tensor Cores, widen instructions, reduce non-matmul ALU work).
   - Both < 50% → **Latency-bound** (check grid size, occupancy, warp stalls, and barriers).
2. **Warp State Statistics (Stall Reasons)**: Identify the #1 stall reason preventing warps from issuing instructions every cycle:

| Top Warp Stall Reason | Root Cause | First Fix to Try |
| :--- | :--- | :--- |
| `Long Scoreboard` | Waiting on global HBM / L2 loads (`LDG`) | Prefetch with `cp.async` / TMA, widen to `float4`, increase occupancy |
| `Short Scoreboard` | Waiting on shared memory (`LDS`) or special math (`MUFU`) | Eliminate bank conflicts, use `ldmatrix`, reduce `expf` calls |
| `Barrier` | Warps idle at `__syncthreads()` | Balance work across warps or adopt warp specialization |
| `MIO Throttle` | Shared-memory or math-pipe instruction queue full | Increase register tiling (`8x8`) so fewer `LDS` ops run per FMA |
| `LG Throttle` | Local/global memory instruction queue saturated | Coalesce loads, eliminate register spills to local memory |

3. **Memory Workload Analysis**: Check L1/TEX and L2 hit rates, global load efficiency (bytes requested vs. sectors transferred — 100% means perfect coalescing), and shared-memory bank conflicts.
4. **Source View (with `-lineinfo`)**: Correlate stalls and SASS instructions (`HMMA`, `LDG.E.128`, `LDS.128`, `STL`/`LDL` spills) directly to your CUDA or Triton source lines.

## Essential `ncu` CLI recipes

```bash
# 1. Profile a single warmed-up kernel launch (skip 5 warmup runs)
ncu --set full \
    --kernel-name regex:my_gemm \
    --launch-skip 5 --launch-count 1 \
    -o gemm_profile ./bench

# 2. Fast CLI summary without saving a huge report (ideal for CI & AI agents)
ncu --metrics \
    sm__throughput.avg.pct_of_peak_sustained_elapsed,\
    gpu__dram_throughput.avg.pct_of_peak_sustained_elapsed,\
    l1tex__data_bank_conflicts_pipe_lsu_mem_shared_op_ld.sum,\
    smsp__warp_issue_stalled_long_scoreboard_per_warp_active.pct \
    --csv --launch-skip 5 --launch-count 1 ./bench

# 3. Profile a PyTorch / Triton script (disable cache flushing only when comparing L2 hits)
ncu --set roofline --kernel-name regex:triton_ -o triton_rep python3 bench_triton.py
```

> [!WARNING]
> **Avoid profiling debug builds or cold launches**
>
> Never profile `-G` builds or launch #0 (which includes JIT compilation and cold caches). Always use `-O3 -lineinfo` and `--launch-skip` to capture steady-state hardware behavior.

## Hands-on lab: parse `ncu --csv` output into a structured bottleneck summary

```python
"""parse_ncu.py — Turn ncu --csv output into an actionable bottleneck verdict."""
import csv, io

SAMPLE_CSV = """"Kernel Name","Metric Name","Metric Value"
"warptile_gemm","sm__throughput.avg.pct_of_peak_sustained_elapsed","84.2"
"warptile_gemm","gpu__dram_throughput.avg.pct_of_peak_sustained_elapsed","31.5"
"warptile_gemm","l1tex__data_bank_conflicts_pipe_lsu_mem_shared_op_ld.sum","0"
"""

metrics = {}
for row in csv.DictReader(io.StringIO(SAMPLE_CSV)):
    metrics[row["Metric Name"]] = float(row["Metric Value"])

sm = metrics["sm__throughput.avg.pct_of_peak_sustained_elapsed"]
dram = metrics["gpu__dram_throughput.avg.pct_of_peak_sustained_elapsed"]
conflicts = int(metrics["l1tex__data_bank_conflicts_pipe_lsu_mem_shared_op_ld.sum"])

regime = "COMPUTE-BOUND" if sm > 70 else ("MEMORY-BOUND" if dram > 70 else "LATENCY-BOUND")
print(f"Regime: {regime} | SM={sm:.1f}% | DRAM={dram:.1f}% | SMEM bank conflicts={conflicts}")
```

## Summary

- Start every `ncu` session with **Speed of Light** to classify compute-bound vs. memory-bound vs. latency-bound, then read **Warp Stall Reasons** to pick the exact code change.
- Use `--launch-skip`, `--launch-count 1`, and `--csv` to integrate Nsight Compute directly into CI regressions and automated kernel-tuning loops.

## Sources

- [NVIDIA Nsight Compute CLI Documentation](https://docs.nvidia.com/nsight-compute/NsightComputeCli/index.html)
- [NVIDIA Kernel Profiling Guide](https://docs.nvidia.com/nsight-compute/ProfilingGuide/index.html)
