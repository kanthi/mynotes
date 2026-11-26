---
title: Triton → CUTLASS → CuTe-DSL
---

# Triton → CUTLASS → CuTe-DSL

Modern GPU kernel authoring spans a clear abstraction ladder: **Triton** for rapid, high-productivity tile programming in Python; **CUTLASS / CuTe C++** for full silicon control over Hopper/Blackwell hardware pipelines; and **CuTe-DSL** (Python front-end over CuTe's layout algebra and MLIR lowering) for combining Python ergonomics with near-100% hardware peak.

## Choosing the right abstraction layer

| Framework | Programming granularity | Strengths | When to drop lower |
| :--- | :--- | :--- | :--- |
| **OpenAI Triton** | Block/tile level in Python | Auto-coalescing, auto-SMEM, fast iteration, huge PyTorch ecosystem | When you need warp specialization, custom TMA/mbarrier rings, or Blackwell TMEM |
| **CUTLASS 3.x / CuTe (C++)** | Thread, warp, warp-group, & cluster | Exact layout algebra, TMA, WGMMA, `tcgen05`, maximum TFLOP/s | Steep C++ template compile times and verbose boilerplate |
| **CuTe-DSL (Python)** | Explicit CuTe layouts + Python syntax | Sub-second JIT compile, direct access to Hopper/Blackwell MMA & TMEM | Still newer ecosystem than Triton for non-NVIDIA targets |

## OpenAI Triton: fused row-wise Softmax example

Beyond trivial vector addition, Triton shines on **memory-bound fused ops** (Softmax, LayerNorm, RMSNorm, RoPE, fused cross-entropy) where writing one clean tile kernel matches or beats hand-written CUDA:

```python
import triton
import triton.language as tl

@triton.jit
def fused_softmax_kernel(
    output_ptr, input_ptr, input_row_stride, output_row_stride, n_cols,
    BLOCK_SIZE: tl.constexpr,
):
    row_idx = tl.program_id(0)
    col_offsets = tl.arange(0, BLOCK_SIZE)
    mask = col_offsets < n_cols

    row_start = input_ptr + row_idx * input_row_stride
    row = tl.load(row_start + col_offsets, mask=mask, other=-float("inf"))

    row_minus_max = row - tl.max(row, axis=0)
    numerator = tl.exp(row_minus_max)
    softmax_out = numerator / tl.sum(numerator, axis=0)

    out_start = output_ptr + row_idx * output_row_stride
    tl.store(out_start + col_offsets, softmax_out, mask=mask)
```

## CuTe Layout Algebra: $\text{Layout} = (\text{Shape}, \text{Stride})$

CUTLASS 3.x unifies global memory, shared memory, swizzled layouts, and Tensor Core register fragments under one mathematical object: a **Layout** mapping logical coordinates to 1D memory offsets via inner product with a stride tuple:

$$\text{Layout} = (\text{Shape}, \text{Stride}), \qquad f(c_0, c_1) = c_0 \cdot s_0 + c_1 \cdot s_1$$

```cpp
#include <cute/tensor.hpp>
using namespace cute;

// 8x16 row-major tile vs. 8x16 column-major tile
auto row_major = make_layout(make_shape(Int<8>{}, Int<16>{}), make_stride(Int<16>{}, Int<1>{}));
auto col_major = make_layout(make_shape(Int<8>{}, Int<16>{}), make_stride(Int<1>{}, Int<8>{}));

// Tile an M x N matrix into 128 x 128 CTA blocks in one algebraic call:
// auto tiled_A = local_tile(mA, Shape<_128, _64>{}, make_coord(blockIdx.x, _));
```

## Hands-on lab: verify CuTe layout coordinate mapping in Python

```python
def cute_offset(coord: tuple[int, ...], stride: tuple[int, ...]) -> int:
    return sum(c * s for c, s in zip(coord, stride))

shape = (4, 8)
row_stride = (8, 1)
col_stride = (1, 4)

for c in [(0, 0), (1, 3), (3, 7)]:
    print(f"coord {c} -> row_major offset={cute_offset(c, row_stride):2d} | col_major offset={cute_offset(c, col_stride):2d}")
```

## Summary

- Start with **Triton** for memory-bound fused ops and rapid prototyping; move to **CuTe-DSL** or **CUTLASS C++** when you need explicit warp specialization, TMA swizzling, or Blackwell `tcgen05` / TMEM control.

## Sources

- [OpenAI Triton Tutorials](https://triton-lang.org/main/getting-started/tutorials/)
- [NVIDIA CUTLASS CuTe Layout Documentation](https://github.com/NVIDIA/cutlass/blob/main/media/docs/cute/01_layout.md)
