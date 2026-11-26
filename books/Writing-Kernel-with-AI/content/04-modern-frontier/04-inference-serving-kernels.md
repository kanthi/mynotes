---
title: Inference-Serving Kernels
---

# Inference-Serving Kernels

LLM inference splits into two radically different physical regimes, which is why serving engines (vLLM, SGLang, TensorRT-LLM) dispatch completely different kernels for each phase:

| Phase | Shape of $Q$ | Arithmetic Intensity | Hardware Bottleneck | Key Kernel Techniques |
| :--- | :--- | :--- | :--- | :--- |
| **Prefill** | $[B \cdot L_{\text{prompt}}, H, D]$ | High (hundreds of FLOPs/byte) | **Compute-bound** (Tensor Cores) | FlashAttention-2/3/4, chunked prefill, FP8/NVFP4 GEMM |
| **Decode** | $[B, 1, H, D]$ | Low (~1–16 FLOPs/byte at small $B$) | **HBM bandwidth-bound** | PagedAttention, Flash-Decoding (Split-KV), CUDA Graphs, W4A16/FP8 weight-only GEMV |

## Why KV cache fragmentation kills serving throughput

In naive serving, each request pre-allocates a contiguous GPU buffer for `max_model_len` tokens (e.g., 32,768 tokens), even if the request finishes in 400 tokens. That wastes **60–80% of GPU HBM** on internal fragmentation and caps maximum batch size $B$ — directly starving decode throughput.

**PagedAttention** (vLLM) applies OS virtual-memory paging to KV caches:
- Physical HBM is carved into fixed-size **KV blocks** (e.g., 16 tokens per block).
- Each sequence holds a lightweight **block table** (`block_tables[seq_id][logical_block] -> physical_block_id`).
- Prefix caching (RadixAttention / Automatic Prefix Caching) shares identical system-prompt blocks across requests with zero duplication (copy-on-write).

## Hands-on lab: simulate PagedAttention block-table indexing and HBM savings

```python
def compare_kv_memory(seq_lengths: list[int], max_len: int = 8192, block_size: int = 16,
                      bytes_per_token: int = 131072):  # ~128 KB/token for 70B GQA model
    naive_tokens = len(seq_lengths) * max_len
    paged_blocks = sum((s + block_size - 1) // block_size for s in seq_lengths)
    paged_tokens = paged_blocks * block_size

    naive_gb = naive_tokens * bytes_per_token / 1e9
    paged_gb = paged_tokens * bytes_per_token / 1e9
    print(f"Naive contiguous KV : {naive_gb:6.2f} GB ({naive_tokens:,} reserved slots)")
    print(f"PagedAttention KV   : {paged_gb:6.2f} GB ({paged_blocks:,} blocks, {paged_gb/naive_gb*100:.1f}% of naive)")

compare_kv_memory([320, 512, 1200, 180, 95, 2400, 640, 410])
```

## Key serving kernels covered in the following chapters

- **Decode Batch-Size Roofline** (`04b`): Computing the exact batch size where decode transitions from memory-bound to compute-bound.
- **Sort-Free Top-p Sampling** (`04c`): Avoiding full-vocabulary $O(V \log V)$ sorts during token sampling.
- **PagedAttention & Hybrid Sliding Window** (`07`, `07b`, `07c`): Block tables, prefix sharing, and eviction policies.
- **Flash-Decoding (Split-KV)** (`08`): Parallelizing single-query decode across the KV sequence dimension so all 132 H100 SMs stay saturated.

## Summary

- Prefill is compute-bound; decode is HBM-bandwidth-bound until batch size reaches the hardware ridge point.
- PagedAttention eliminates contiguous KV fragmentation via 16-token virtual-to-physical block tables, unlocking 2–4x higher serving concurrency.

## Sources

- [Kwon et al., Efficient Memory Management for Large Language Model Serving with PagedAttention (SOSP 2023)](https://arxiv.org/abs/2309.06180)
