---
title: FlashAttention from Scratch (FA1 → FA2 → FA3)
---

# FlashAttention from Scratch (FA1 → FA2 → FA3)

FlashAttention evolved across three major generations — each one removing the bottleneck exposed by the previous generation's speedup. Understanding the exact algorithmic and hardware shifts from FA1 to FA2 to FA3 is the fastest way to see how modern GPU kernel design tracks silicon evolution from Ampere (SM80) to Hopper (SM90).

## Generational comparison: FA1 vs. FA2 vs. FA3

| Dimension | FlashAttention-1 (2022) | FlashAttention-2 (2023) | FlashAttention-3 (2024) |
| :--- | :--- | :--- | :--- |
| **Target silicon** | Ampere (A100) | Ampere / Ada | Hopper (H100, SM90) |
| **Loop order** | Outer: $K, V$; Inner: $Q$ (extra HBM writes of $O$) | **Outer: $Q$; Inner: $K, V$** (write $O_i$ once!) | Outer: $Q$ (persistent CTAs); Inner: async TMA $K, V$ |
| **Parallelism** | Batch $\times$ Heads only | Batch $\times$ Heads $\times$ **Sequence length** | Persistent warp-specialized CTAs + LPT scheduling |
| **Warp partitioning** | Split-$K$ across warps (extra SMEM reductions) | **Split-$Q$ across warps** (zero inter-warp SMEM sync) | Producer (TMA) + Consumer (WGMMA) warp groups |
| **Precision & peak %** | FP16/BF16 (~25–40% A100) | FP16/BF16 (~50–73% A100) | FP16 (~75% H100) + **block-quantized FP8** (~1.2 PFLOP/s) |

## Why swapping the loops in FA2 doubled speed

In FA1, iterating over $K_j, V_j$ in the outer loop forced each inner $Q_i$ step to read $O_i$ from HBM, rescale it, and write $O_i$ back to HBM. Swapping to **outer $Q_i$, inner $K_j, V_j$** keeps $O_i$ completely resident in thread registers across the entire inner loop and writes $O_i$ to HBM **exactly once**. Deferring the division by $l_i$ until the very end of the inner loop also cuts non-matmul FLOPs dramatically.

## Minimal FlashAttention-2 forward kernel in Triton

```python
import triton
import triton.language as tl

@triton.jit
def flash_attn_v2_fwd(
    Q, K, V, Out,
    stride_qz, stride_qh, stride_qm, stride_qk,
    stride_kz, stride_kh, stride_kn, stride_kk,
    stride_vz, stride_vh, stride_vn, stride_vk,
    stride_oz, stride_oh, stride_om, stride_ok,
    N_CTX, sm_scale,
    BLOCK_M: tl.constexpr, BLOCK_N: tl.constexpr, HEAD_DIM: tl.constexpr,
):
    pid_m = tl.program_id(0)   # Sequence-parallel Q tile
    pid_hz = tl.program_id(1)  # Batch * Head index

    offs_m = pid_m * BLOCK_M + tl.arange(0, BLOCK_M)
    offs_n = tl.arange(0, BLOCK_N)
    offs_d = tl.arange(0, HEAD_DIM)

    q_ptrs = Q + pid_hz * stride_qh + offs_m[:, None] * stride_qm + offs_d[None, :] * stride_qk
    q = tl.load(q_ptrs, mask=offs_m[:, None] < N_CTX, other=0.0)

    m_i = tl.full([BLOCK_M], -float("inf"), dtype=tl.float32)
    l_i = tl.zeros([BLOCK_M], dtype=tl.float32)
    acc = tl.zeros([BLOCK_M, HEAD_DIM], dtype=tl.float32)

    # Inner loop streams K, V blocks while Q and acc stay in SRAM/registers
    for start_n in range(0, N_CTX, BLOCK_N):
        cols = start_n + offs_n
        k_ptrs = K + pid_hz * stride_kh + cols[:, None] * stride_kn + offs_d[None, :] * stride_kk
        v_ptrs = V + pid_hz * stride_vh + cols[:, None] * stride_vn + offs_d[None, :] * stride_vk
        k = tl.load(k_ptrs, mask=cols[:, None] < N_CTX, other=0.0)
        v = tl.load(v_ptrs, mask=cols[:, None] < N_CTX, other=0.0)

        qk = tl.dot(q, tl.trans(k)) * sm_scale
        qk = tl.where(cols[None, :] < N_CTX, qk, -float("inf"))

        m_ij = tl.maximum(m_i, tl.max(qk, axis=1))
        p = tl.exp(qk - m_ij[:, None])
        alpha = tl.exp(m_i - m_ij)
        l_i = l_i * alpha + tl.sum(p, axis=1)
        acc = acc * alpha[:, None] + tl.dot(p.to(v.dtype), v)
        m_i = m_ij

    acc = acc / l_i[:, None]
    o_ptrs = Out + pid_hz * stride_oh + offs_m[:, None] * stride_om + offs_d[None, :] * stride_ok
    tl.store(o_ptrs, acc.to(Out.dtype.element_ty), mask=offs_m[:, None] < N_CTX)
```

## What FlashAttention-3 adds on Hopper (H100)

- **Warp-specialized producer/consumer pipeline**: Producer warps issue async TMA loads while consumer warp groups execute WGMMA (`gemm0` for $QK^T$ and `gemm1` for $PV$).
- **Ping-pong scheduling**: Two consumer warp groups alternate between Softmax (SFU/MUFU) and WGMMA (Tensor Cores) so the exponential unit never stalls the Tensor Cores.
- **Incoherent processing for FP8**: Random orthogonal (Hadamard) rotations spread outlier channels before FP8 quantization, cutting FP8 attention RMSE by ~2.6x.

## Summary

- FA1 eliminated $O(N^2)$ HBM memory; FA2 swapped loops and split $Q$ across warps to saturate Ampere Tensor Cores; FA3 introduced Hopper TMA, WGMMA ping-ponging, and low-error FP8.

## Sources

- [Dao, FlashAttention-2: Faster Attention with Better Parallelism and Work Partitioning (2023)](https://arxiv.org/abs/2307.08691)
- [Shah et al., FlashAttention-3: Fast and Accurate Attention with Asynchrony and Low-precision (2024)](https://arxiv.org/abs/2407.08608)
