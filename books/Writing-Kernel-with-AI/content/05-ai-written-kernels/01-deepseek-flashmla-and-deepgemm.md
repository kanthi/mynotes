---
title: 'DeepSeek: FlashMLA & DeepGEMM'
---

# DeepSeek: FlashMLA & DeepGEMM

DeepSeek demonstrated that compact, readable, purpose-built GPU kernels can outperform sprawling general-purpose vendor libraries on real frontier-model workloads. Two open-source releases — **FlashMLA** and **DeepGEMM** — became instant blueprints for modern Hopper kernel engineering.

## 1. FlashMLA: Decoding Multi-Head Latent Attention (MLA)

In standard Grouped-Query Attention (GQA), the KV cache stores separate $K$ and $V$ tensors per KV head. DeepSeek-V2/V3/R1 replace this with **Multi-Head Latent Attention (MLA)**: every token's KV state is compressed into a single low-rank latent vector $c_t^{KV} \in \mathbb{R}^{512}$ plus a 64-dim decoupled RoPE key $k_t^R$ — **576 elements total per token** regardless of head count, cutting KV cache memory by >90% vs. MHA.

During decode, instead of materializing full $K$ and $V$ matrices in HBM, FlashMLA:
1. Absorbs the up-projection weight $W^{UK}$ directly into the query vector ($q' = q W^{UK}$), so attention scores are computed directly against the compressed latent vector $c_t^{KV}$!
2. Streams paged 64-token latent blocks from HBM using Hopper TMA and Split-KV parallelism across all 132 SMs.
3. Saturates H100 HBM bandwidth (~3,000 GB/s) and up to ~580 TFLOP/s on variable-length decode batches.

## 2. DeepGEMM: ~300 Lines of Clean FP8 GEMM with Fine-Grained Scaling

Standard Hopper FP8 WGMMA accumulates inside low-precision internal accumulators (~14-bit mantissa) before adding to FP32 registers, which causes accuracy loss on long $K$ reductions unless promoted frequently. **DeepGEMM** solves this in ~300 lines of modular CUDA + lightweight JIT specialization:

- **Fine-grained 2D/1D block scaling**: Activations use $1 \times 128$ per-token group scales; weights use $128 \times 128$ block scales.
- **Two-level FP32 promotion**: Accumulates 4 WGMMA steps in native accumulators, then promotes into dedicated FP32 registers with per-block scaling factors (`FFMA`) to restore full FP32 accuracy.
- **Zero C++ template bloat**: Uses a tiny Python JIT wrapper that bakes exact `M, N, K` shapes and TMA descriptors at runtime, compiling in milliseconds and matching or beating cuBLASLt on DeepSeek-V3 MoE grouped GEMMs.

## Hands-on lab: verify the MLA matrix-absorption identity in PyTorch

Run this 15-line PyTorch check to prove that absorbing $W^{UK}$ into $Q$ produces the exact same attention logits as explicitly decompressing $K = C_{KV} W^{UK}$ — without ever writing full $K$ to memory:

```python
import torch

torch.manual_seed(0)
B, T, d_c, d_h = 2, 128, 512, 128
Q = torch.randn(B, 1, d_h)          # Single decode query
C_kv = torch.randn(B, T, d_c)       # Compressed KV latent cache
W_uk = torch.randn(d_c, d_h)        # Up-projection matrix

# Naive: decompress K (expensive HBM write/read of T x d_h)
K_full = C_kv @ W_uk
logits_naive = Q @ K_full.transpose(-1, -2)

# FlashMLA absorption: fold W_uk into Q once, dot directly with C_kv!
Q_absorbed = Q @ W_uk.T             # [B, 1, d_c]
logits_absorbed = Q_absorbed @ C_kv.transpose(-1, -2)

print("Max abs diff (absorbed vs decompressed):", (logits_naive - logits_absorbed).abs().max().item())
```

## Summary

- **FlashMLA** absorbs KV up-projections into queries so decode attention reads only compact 576-dim latent vectors per token from HBM.
- **DeepGEMM** proves that ~300 lines of JIT-specialized Hopper CUDA with fine-grained $1 \times 128$ / $128 \times 128$ FP8 scaling and two-level FP32 promotion can rival or beat monolithic vendor GEMM libraries.

## Sources

- [DeepSeek FlashMLA Repository](https://github.com/deepseek-ai/FlashMLA)
- [DeepSeek DeepGEMM Repository](https://github.com/deepseek-ai/DeepGEMM)
