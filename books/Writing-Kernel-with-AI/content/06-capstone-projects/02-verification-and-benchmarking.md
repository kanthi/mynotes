---
title: Verification, Profiling, and Benchmarking Harness
---

# Verification, Profiling, and Benchmarking Harness

A fast kernel that silently produces NaNs on non-aligned shapes — or looks 2x faster only because its input sat hot in L2 cache — is not shippable. Every capstone submission must pass a **three-tiered validation and benchmarking harness**: numerical parity across edge cases, L2-flushed CUDA-event timing, and Nsight Compute hardware-counter verification.

## Tier 1: Multi-shape numerical oracle

Compare your kernel against an FP64 reference across power-of-two shapes, prime/unaligned dimensions (`31, 33, 127, 4097`), extreme magnitudes ($10^{-5}$ to $10^{4}$), and NaN/Inf guards:

$$\frac{\|Y_{\text{kernel}} - Y_{\text{ref}}\|_2}{\|Y_{\text{ref}}\|_2} < 10^{-3} \quad (\text{BF16/FP16}), \qquad < 10^{-5} \quad (\text{FP32})$$

```python
import torch

def verify_kernel(kernel_fn, ref_fn, shapes, dtype=torch.bfloat16, max_rel_l2=1e-3):
    for shape in shapes:
        x = torch.randn(*shape, device="cuda", dtype=dtype)
        y_ref = ref_fn(x.double()).to(torch.float64)
        y_out = kernel_fn(x).to(torch.float64)
        assert torch.isfinite(y_out).all(), f"Non-finite output on shape {shape}"
        rel_l2 = (y_out - y_ref).norm() / y_ref.norm().clamp(min=1e-12)
        assert rel_l2 < max_rel_l2, f"Shape {shape}: rel_l2={rel_l2:.3e} >= {max_rel_l2}"
        print(f"PASS {str(shape):18s} | rel_l2={rel_l2:.2e}")
```

## Tier 2: L2-cache-flushed CUDA Event benchmarking

For memory-bound kernels, looping the exact same small tensor 1,000 times measures **L2 cache bandwidth (~10 TB/s)** instead of real **HBM bandwidth (~3.35 TB/s)**. Flush the L2 cache between timed launches (using a dummy 256 MB buffer) and report median + IQR latency:

```python
import statistics
import torch

def bench_cuda_events(fn, warmup=25, iters=100, flush_l2=True):
    l2_buf = torch.empty(int(256e6 // 4), dtype=torch.int32, device="cuda") if flush_l2 else None
    for _ in range(warmup):
        fn()
    torch.cuda.synchronize()

    times_ms = []
    for _ in range(iters):
        if l2_buf is not None:
            l2_buf.zero_()  # Evict previous inputs from L2 cache
        start = torch.cuda.Event(enable_timing=True)
        end = torch.cuda.Event(enable_timing=True)
        start.record()
        fn()
        end.record()
        torch.cuda.synchronize()
        times_ms.append(start.elapsed_time(end))

    return statistics.median(times_ms), min(times_ms)
```

### Equivalent C++ CUDA Event timing loop

```cpp
cudaEvent_t start, stop;
cudaEventCreate(&start);
cudaEventCreate(&stop);

for (int i = 0; i < 20; ++i) launch_kernel<<<grid, block>>>(args...); // warmup
cudaDeviceSynchronize();

cudaEventRecord(start);
for (int i = 0; i < 200; ++i) launch_kernel<<<grid, block>>>(args...);
cudaEventRecord(stop);
cudaEventSynchronize(stop);

float total_ms = 0.0f;
cudaEventElapsedTime(&total_ms, start, stop);
printf("Avg kernel time: %.4f ms\n", total_ms / 200.0f);
```

## Tier 3: Sanitizer & Nsight Compute sign-off

```bash
compute-sanitizer --tool memcheck python3 tests/test_correctness.py
compute-sanitizer --tool racecheck python3 tests/test_correctness.py
ncu --set full --launch-skip 10 --launch-count 1 -o capstone_rep python3 benchmarks/bench_roofline.py
```

## Summary

- Validate against FP64 references across unaligned shapes and extreme scales before measuring speed.
- Flush L2 cache between timed CUDA Event launches for memory-bound kernels, report median latency, and confirm zero errors under `compute-sanitizer`.
