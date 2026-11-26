---
title: 'LLM-Generated Kernels: The Agent + Profiler Loop'
---

# LLM-Generated Kernels: The Agent + Profiler Loop

Writing a peak-speed GPU kernel manually takes days of iterating across tile sizes, swizzle layouts, register budgets, and Nsight Compute stall reports. **Autonomous kernel agents** (such as Google DeepMind's **AlphaEvolve**, **KernelBench** pipelines, and hardware-guided evolutionary loops) automate that tight inner loop by pairing an LLM code generator with a real compiler, a strict numerical oracle, and hardware profiler telemetry.

![Closed-loop autonomous kernel optimization showing specification and hardware constraints fed to an LLM kernel agent, verified against a numerical oracle and anti-reward-hacking sanitizer, profiled with Nsight Compute, and fed back through a candidate archive.](../../images/diagram-agent-profiler-loop.svg)

## Anatomy of a reliable agent-profiler loop

Every production-grade kernel-synthesis loop enforces four strict stages on every candidate mutation:

1. **Compile & static audit**: Rejects syntax errors, disallowed imports/fallbacks (e.g., sneaking in `torch.matmul`), and `ptxas` register spills.
2. **Multi-shape numerical oracle**: Evaluates randomized, non-aligned, edge-case shapes against an FP64/PyTorch reference in an **isolated subprocess** (preventing reward hacking via monkey-patching or cached outputs, covered in Chapter 4 of this part).
3. **Warm CUDA-event benchmark + `ncu` counters**: Measures median latency over warmed runs and extracts exact bottleneck counters (`SM%`, `DRAM%`, `bank_conflicts`, top warp stall reason).
4. **Evolutionary archive update**: Stores `(code, latency_us, ncu_summary)` in a population archive and prompts the next generation with concrete profiler deltas (*"v3 hit 32-way SMEM bank conflicts on line 44; apply XOR swizzling"*).

## Minimal Python skeleton for a closed-loop kernel harness

```python
from dataclasses import dataclass
import subprocess, json

@dataclass
class EvalResult:
    compiled: bool
    correct: bool
    median_us: float | None
    ncu_summary: str
    error: str = ""

def evaluate_candidate(candidate_path: str, ref_spec: str) -> EvalResult:
    """Run candidate kernel in an isolated subprocess so it cannot tamper with the oracle."""
    proc = subprocess.run(
        ["python3", "isolated_verifier.py", "--kernel", candidate_path, "--spec", ref_spec],
        capture_output=True, text=True, timeout=60,
    )
    if proc.returncode != 0:
        return EvalResult(False, False, None, "", error=proc.stderr[-1000:])
    data = json.loads(proc.stdout)
    return EvalResult(
        compiled=True,
        correct=data["correct"],
        median_us=data["median_us"],
        ncu_summary=data["ncu_summary"],
    )
```

## Google DeepMind AlphaEvolve & KernelBench

- **KernelBench** (Stanford): Standardizes evaluation across Level 1 (single ops), Level 2 (fused operator chains), and Level 3 (full model blocks), scoring `fast_p` (fraction of kernels that are both **correct** and at least $p\times$ faster than eager PyTorch).
- **AlphaEvolve**: Pairs evolutionary program databases with Gemini models and real hardware evaluators, discovering non-obvious tiling and scheduling improvements (such as faster FlashAttention variants and 4x4 complex matrix multiplication algorithms) that beat hand-tuned baselines.

## The evolving role of the human kernel engineer

Agents excel at sweeping tile dimensions, unroll factors, and instruction schedules; humans own the **specification and verification boundary**:
- Writing airtight, tamper-proof numerical oracles and anti-reward-hacking harnesses.
- Supplying the exact hardware card (SMEM size, bank width, MMA shapes, precision rules).
- Reading Nsight Compute reports when the agent gets stuck in a local minimum (e.g., switching from block-level tiling to warp specialization).

## Summary

- Autonomous kernel loops succeed only when compiler diagnostics, isolated numerical verification, and Nsight Compute hardware counters are fed directly back into the agent's context.
- Never optimize raw wall-clock speed without an isolated verifier and strict precision tolerances.

## Sources

- [Ouyang et al., KernelBench: Can LLMs Write Efficient GPU Kernels? (2025)](https://arxiv.org/abs/2502.10517)
- [Google DeepMind — AlphaEvolve: A Gemini-Powered Coding Agent for Designing Advanced Algorithms](https://deepmind.google/discover/blog/alphaevolve-a-gemini-powered-coding-agent-for-designing-advanced-algorithms/)
