---
title: Crusoe Capstone Specifications & Problem Tracks
---

# Crusoe Capstone Specifications & Problem Tracks

The capstone project asks you to design, verify, profile, and defend a production-grade GPU kernel (CUDA, Triton, or CuTe-DSL) that solves a real inference or training bottleneck — complete with a `torch.library` custom-op wrapper, `compute-sanitizer` verification, Nsight Compute Roofline analysis, and a reproducible benchmark suite.

## Problem tracks and quantitative success criteria

| Track | Core Kernel | Target Regime | Minimum Speedup Target | Key Technical Challenge |
| :--- | :--- | :--- | :--- | :--- |
| **Track 1: Fused Block-Sparse Attention** | Block-sparse FlashAttention (sliding window + global sinks) | Long context ($N \ge 16,384$, $d = 128$, BF16) | $\ge 2.0\times$ vs. dense FlashAttention / SDPA | Skipping zero blocks cleanly via CSR block masks while preserving online softmax & LPT load balancing |
| **Track 2: W4A16 / FP8 Quantized GEMM** | Weight-only INT4/NVFP4 GEMM with fused register dequantization | Decode ($M \in [1, 64]$, $N, K \in [4096, 14336]$) | $\ge 2.5\times$ vs. BF16 cuBLAS at $M \le 16$ | Coalesced packed-bit loads, fast bitwise/LUT unpack in registers, Split-K for small $M$ |
| **Track 3: Fused Speculative Verification** | Batched tree/draft token verification + rejection sampling | Serving ($B \in [1, 128]$, $K = 4\text{--}8$ drafts, $V = 128\text{k}$) | $\ge 3.0\times$ vs. eager PyTorch verification | Fusing softmax, target/draft probability ratio, and bonus-token sampling in a single HBM pass |

## Required repository layout

```text
capstone-kernel/
├── csrc/ or triton_ops/   # Kernel source + torch.library custom-op registration
├── tests/
│   ├── test_correctness.py # Multi-shape FP64 reference parity + edge cases
│   └── test_compile.py     # torch.compile(fullgraph=True) + CUDA Graph compatibility
├── benchmarks/
│   ├── bench_roofline.py   # Sweeps shapes & plots TFLOP/s + GB/s vs hardware ceiling
│   └── ncu_profile.sh      # Reproducible Nsight Compute command + saved summary
└── README.md               # Architecture write-up, Roofline chart, and speedup table
```

## Milestone schedule

1. **Milestone 1 (Correctness baseline)**: Working reference PyTorch op + naive kernel passing all edge-case shape tests (`N=1, 31, 32, 33, 4097`).
2. **Milestone 2 (Roofline & bottleneck diagnosis)**: First `ncu` report identifying the exact limiter (`Long Scoreboard`, `MIO Throttle`, bank conflicts, or occupancy).
3. **Milestone 3 (Production optimization)**: Vectorized loads, swizzled SMEM/registers, and optional agent-guided tuning beating the target baseline.
4. **Milestone 4 (Packaging & defense)**: Clean `torch.compile` integration, reproducible benchmark script, and Demo Day presentation.

## Summary

- Choose one of the three production tracks, lock down an FP64 numerical oracle on day one, and let Nsight Compute guide every optimization step through Demo Day.
