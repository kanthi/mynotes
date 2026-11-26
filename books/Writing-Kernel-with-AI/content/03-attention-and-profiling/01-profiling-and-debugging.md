---
title: Profiling & Debugging Like a Pro
---

# Profiling & Debugging Like a Pro

Optimizing a GPU kernel without measurements is guesswork. A kernel that looks "clever" in source code can be silently spilling 40 registers per thread to local DRAM, serializing every shared-memory read across 32-way bank conflicts, or launching tiny grids that leave 80% of the GPU idle. Professional kernel engineers follow a strict diagnostic ladder: **correctness first (`compute-sanitizer`), timeline second (`nsys`), and microarchitecture third (`ncu`)**.

## The three-tool NVIDIA diagnostic stack

| Tool | Scope | What it catches | Typical command |
| :--- | :--- | :--- | :--- |
| **`compute-sanitizer`** | Functional correctness | Out-of-bounds global/SMEM reads, uninitialized memory, shared-memory race conditions | `compute-sanitizer --tool memcheck ./app` |
| **Nsight Systems (`nsys`)** | Whole-app timeline | CPU launch bubbles, H2D/D2H sync stalls, stream serialization, Python overhead | `nsys profile -o trace --stats=true ./app` |
| **Nsight Compute (`ncu`)** | Single kernel internals | Roofline position, warp stall reasons, bank conflicts, register spills, L2 hit rate | `ncu --set full --kernel-name regex:gemm -o rep ./app` |

> [!TIP]
> **Compile with `-lineinfo` (not `-G`) when profiling**
>
> `-G` disables compiler optimizations and turns every register variable into memory traffic, producing completely meaningless timings. Always profile `-O3 -lineinfo` so Nsight Compute maps every warp stall directly back to your CUDA source line at full release speed.

## Three classic sabotaged-kernel incidents (and exact fixes)

### Incident 1: Silent shared-memory race condition

Threads write into shared memory and neighboring threads read it without a barrier:

```cuda
// BUG: Thread t+1 may read smem[t+1] before thread t+1 writes it (or after next iter)
smem[threadIdx.x] = input[idx];
float right = smem[threadIdx.x + 1];  // Race condition!

// FIX: Always separate shared-memory write and cross-thread read with __syncthreads()
smem[threadIdx.x] = input[idx];
__syncthreads();
float right = (threadIdx.x + 1 < blockDim.x) ? smem[threadIdx.x + 1] : 0.0f;
```

- **Detect**: `compute-sanitizer --tool racecheck ./my_kernel` (also run `--tool initcheck` for uninitialized reads and `--tool synccheck` for divergent `__syncthreads()`).

### Incident 2: 32-way shared-memory bank conflicts

Reading down a column of `__shared__ float tile[32][32]` maps all 32 lanes of a warp to Bank 0, serializing one 4-cycle load into 32 back-to-back cycles.

- **Detect**: NCU metric `l1tex__data_bank_conflicts_pipe_lsu_mem_shared_op_ld.sum`.
- **Fix**: Pad the inner dimension (`__shared__ float tile[32][33]`) or apply XOR swizzling (`tile[row][col ^ row]`).

### Incident 3: Catastrophic register spilling

If your kernel uses 88 registers/thread but `__launch_bounds__` or `--maxrregcount=64` forces a 64-register cap, the compiler spills excess variables into **local memory** (which resides in L1/L2/HBM, 20–100x slower than registers).

- **Detect**: Compile with `nvcc -O3 --ptxas-options=-v` and inspect `spill stores` / `spill loads`:

```text
ptxas info : Used 64 registers, 128 bytes stack frame, 32 bytes spill stores, 32 bytes spill loads
```

- **Fix**: Reduce live accumulator tile size, unroll less aggressively (`#pragma unroll 4` instead of full unroll), or relax `minBlocksPerMultiprocessor` in `__launch_bounds__`.

## Reusable CUDA error-checking macro

Silent CUDA errors are the #1 cause of "why is my output tensor all zeros?" Wrap every runtime API call and check `cudaGetLastError()` immediately after every kernel launch:

```cuda
#define CUDA_CHECK(call)                                                        \
    do {                                                                        \
        cudaError_t err__ = (call);                                             \
        if (err__ != cudaSuccess) {                                             \
            fprintf(stderr, "CUDA error %s:%d: %s\n",                           \
                    __FILE__, __LINE__, cudaGetErrorString(err__));             \
            std::exit(EXIT_FAILURE);                                            \
        }                                                                       \
    } while (0)

// After a kernel launch:
my_kernel<<<grid, block>>>(args...);
CUDA_CHECK(cudaGetLastError());       // Catches invalid grid/block or SMEM limits
CUDA_CHECK(cudaDeviceSynchronize());  // Catches async illegal memory access
```

## Hands-on lab: run the 3-stage diagnostic checklist on any CUDA binary

```bash
# 1. Build with release optimizations + line numbers + register/spill report
nvcc -O3 -lineinfo -arch=sm_80 --ptxas-options=-v kernel.cu -o kernel

# 2. Correctness: check bounds, races, and uninitialized reads
compute-sanitizer --tool memcheck ./kernel
compute-sanitizer --tool racecheck ./kernel
compute-sanitizer --tool initcheck ./kernel

# 3. Timeline: verify no CPU launch gaps or accidental host syncs
nsys profile -o timeline --stats=true ./kernel

# 4. Microarchitecture: capture roofline, stalls, and bank conflicts
ncu --set full --launch-skip 5 --launch-count 1 -o report ./kernel
```

## Summary

- Follow the strict ladder: `compute-sanitizer` (correctness) → `nsys` (timeline) → `ncu` (kernel internals).
- Always compile profiling builds with `-O3 -lineinfo` and `--ptxas-options=-v` to catch register spills immediately.
- Wrap every CUDA call and kernel launch with `CUDA_CHECK` so launch failures never fail silently.

## Sources

- [NVIDIA Compute Sanitizer User Manual](https://docs.nvidia.com/compute-sanitizer/ComputeSanitizer/index.html)
- [NVIDIA Nsight Systems Documentation](https://docs.nvidia.com/nsight-systems/)
- [NVIDIA Nsight Compute Documentation](https://docs.nvidia.com/nsight-compute/)
