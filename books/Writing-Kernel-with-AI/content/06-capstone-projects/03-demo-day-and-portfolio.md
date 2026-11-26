---
title: Demo Day Presentation & Portfolio
---

# Demo Day Presentation & Portfolio

A great kernel engineering portfolio reads like an empirical lab notebook, not a marketing slide deck. Reviewers and hiring managers look for three things immediately: **can I reproduce your numbers in one command**, **did you diagnose the hardware bottleneck before optimizing**, and **can you defend every tile size, precision choice, and trade-off?**

## Deliverables checklist

1. **Reproducible GitHub repository**: Clean `pyproject.toml` / `CMakeLists.txt`, zero hardcoded absolute paths, pinned CUDA/PyTorch/Triton versions, and a one-command `make test && make bench`.
2. **Roofline plot**: Plots your kernel's measured TFLOP/s and arithmetic intensity (FLOPs/byte) against the GPU's theoretical HBM bandwidth roof and Tensor Core compute roof across your full shape sweep.
3. **Progression worklog table**: Shows each optimization step from naive baseline to final kernel with exact hardware counters:

| Version | Key Optimization | Median Time (ms) | TFLOP/s | HBM BW (GB/s) | Top `ncu` Bottleneck |
| :--- | :--- | :---: | :---: | :---: | :--- |
| `v0` | Naive baseline | 4.82 | 28.5 | 410 | Uncoalesced global loads |
| `v1` | SMEM tiling + `float4` | 1.15 | 119.4 | 1,620 | 32-way SMEM bank conflicts |
| `v2` | XOR swizzle + $8 \times 8$ reg tile | 0.34 | 404.1 | 2,180 | `stall_long_scoreboard` |
| `v3` | Async TMA + 4-stage ring | 0.19 | 723.0 | 2,890 | Near Tensor Core peak |

4. **5-minute technical defense**: Walk through (a) the exact problem & Roofline regime, (b) the single `ncu` metric that unlocked the biggest jump, (c) how you prevented numerical drift or reward hacking, and (d) what you would change next for Blackwell (`tcgen05` / TMEM).

## Hands-on lab: generate a publication-ready Roofline plot from your benchmark CSV

```python
import matplotlib.pyplot as plt
import numpy as np

# Example H100 SXM5 BF16 roofline (989 TFLOP/s, 3.35 TB/s HBM)
peak_tflops, peak_bw_tbs = 989.0, 3.35
ai = np.logspace(-1, 3.5, 400)
roof = np.minimum(peak_tflops, ai * peak_bw_tbs)

fig, ax = plt.subplots(figsize=(7, 4.2))
ax.loglog(ai, roof, linewidth=2.2, label="H100 BF16 Roofline")
ax.scatter([12, 64, 240], [34, 195, 723], s=70, zorder=5, label="Capstone Kernel (v1 -> v3)")
ax.set_xlabel("Arithmetic Intensity (FLOPs / Byte)")
ax.set_ylabel("Achieved Throughput (TFLOP/s)")
ax.set_title("Capstone Kernel Roofline Progression")
ax.grid(True, which="both", linestyle="--", alpha=0.4)
ax.legend()
fig.tight_layout()
fig.savefig("roofline_progression.png", dpi=160)
```

## Summary

- Ship a one-command reproducible repository, a step-by-step optimization table backed by `ncu` counters, and a Roofline chart showing how close your final kernel sits to silicon peak.
