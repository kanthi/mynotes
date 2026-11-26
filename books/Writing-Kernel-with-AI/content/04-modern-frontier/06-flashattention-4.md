---
title: Flash Attention 4
---

# Flash Attention 4

On NVIDIA Blackwell (B200), 5th-generation Tensor Cores (`tcgen05`) doubled BF16 matrix-multiply throughput over Hopper — while the **Special Function Unit (SFU / MUFU)** that computes `exp2f` for Softmax did **not** double at the same rate. Suddenly, Softmax's exponential and rescaling step — once a minor 10% overhead on Ampere — became the dominant bottleneck blocking >1 PFLOP/s sustained attention.

## Four innovations that push FlashAttention-4 past 1 PFLOP/s

1. **Software-emulated `exp2` via FMA polynomials**: Offloads a fraction of `exp2` evaluations from the saturated SFU (`MUFU.EX2`) onto idle FP32 FMA units using a degree-3/4 Horner polynomial approximation on the fractional mantissa, keeping both SFU and FMA pipes busy in parallel.
2. **Conditional / Adaptive Softmax rescaling**: When a newly streamed $K_j$ block does not raise the running row maximum $m_i$ beyond a small threshold $\tau$ (meaning $\alpha = \exp(m_{\text{prev}} - m_{\text{new}}) \approx 1$), FA4 skips rescaling the entire $O_i$ accumulator tile in that iteration and folds the correction into the final normalization.
3. **TMEM-resident accumulators + 2-CTA cooperative `tcgen05`**: Stores $S_{ij}$ and $O_i$ accumulators in Blackwell's 256 KB Tensor Memory (TMEM), freeing registers for overlapped Softmax correction warps while `tcgen05` executes the next $Q K^T$ or $P V$ tile.
4. **Authored in Python CuTe-DSL**: Eliminates C++ template compile times (~30x faster iteration) while lowering directly to PTX `tcgen05`, TMA, and TMEM primitives.

## Hands-on lab: verify Horner polynomial `exp2` accuracy vs. `torch.exp2`

```python
import torch

def poly_exp2_frac(f: torch.Tensor) -> torch.Tensor:
    """Degree-3 minimax-style approximation of 2^f for f in [0, 1)."""
    # Evaluates with 3 fused multiply-adds (zero SFU instructions)
    c0, c1, c2, c3 = 1.0, 0.69314718, 0.24022651, 0.05550411
    return ((c3 * f + c2) * f + c1) * f + c0

x = torch.linspace(-10.0, 0.0, 10000)
n = torch.floor(x)
f = x - n
approx = torch.ldexp(poly_exp2_frac(f), n.to(torch.int32))
exact = torch.exp2(x)
print("Max relative error of 3-FMA exp2:", ((approx - exact).abs() / exact).max().item())
```

A 3-FMA polynomial keeps relative error around $10^{-3}$ (and degree-4 reaches $< 10^{-4}$), letting FA4 balance SFU and FMA throughput seamlessly on Blackwell.

## Summary

- On Blackwell, SFU `exp2` throughput lags Tensor Core `tcgen05` throughput, making Softmax the primary bottleneck in attention.
- FlashAttention-4 breaks 1 PFLOP/s by combining TMEM-resident accumulators, conditional rescaling, FMA-emulated `exp2`, and Python CuTe-DSL code generation.

## Sources

- [Tri Dao / Colfax Research — FlashAttention-4 on Blackwell (CuTe-DSL)](https://github.com/Dao-AILab/flash-attention)
