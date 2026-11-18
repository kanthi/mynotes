---
title: Linux Performance Engineering & Server Benchmarking
unnumbered: true
---

# Linux Performance Engineering & Server Benchmarking

Performance is not a switch you flip or a list of magical sysctl parameters copied from internet forums. Performance is an emergent property of the entire system—the interaction between user code, language runtimes, kernel subsystems, hardware microarchitecture, interconnect topologies, and persistent storage media.

This section provides a rigorous, deep-dive curriculum into Linux performance engineering and server benchmarking. Spanning 28 structured chapters, it guides systems engineers, SREs, kernel developers, and infrastructure architects from first principles and statistical measurement methodologies to bare-metal hardware profiling, kernel subsystem internals, eBPF tracing, high-load benchmarking, and production cluster optimization.

```
┌────────────────────────────────────────────────────────────────────────┐
│             LINUX PERFORMANCE & BENCHMARKING TAXONOMY                  │
│                                                                        │
│   Workload / Application Layer                                         │
│   ├─ SLOs / SLIs, Latency Profiles (p99/p99.9), Throughput vs Saturation│
│   └─ Runtimes: JVM, Go runtime, Python, Databases (RDBMS & LSM-trees)  │
│                                │                                       │
│                                ▼                                       │
│   Linux Kernel Subsystems                                              │
│   ├─ CPU Scheduler: CFS/EEVDF, Real-Time (SCHED_FIFO/DEADLINE), Affinity│
│   ├─ Virtual Memory: Page tables, Folios, Slab/Slub, Swapping, THP     │
│   ├─ VFS & Page Cache: Read-ahead, Dirty page writeback, mmap          │
│   ├─ Block Layer: blk-mq, I/O schedulers, Bio queues, NVMe drivers    │
│   └─ Network Stack: NAPI softirqs, RSS/RPS/XPS, TCP BDP & Congestion   │
│                                │                                       │
│                                ▼                                       │
│   Observability & Instrumentation                                      │
│   ├─ Hardware Counters: PMU, IPC, Cache Misses, Branch Mispredictions  │
│   ├─ Tracing: perf, ftrace, bpftrace, BCC, Off-CPU Flame Graphs        │
│   └─ Saturation Metrics: Pressure Stall Information (PSI), sar, vmstat │
│                                │                                       │
│                                ▼                                       │
│   Physical Hardware Microarchitecture                                  │
│   └─ NUMA fabrics (UPI/Infinity Fabric), SMT cores, PCIe Gen5/Gen6, CXL│
└────────────────────────────────────────────────────────────────────────┘
```

## Curriculum Breakdown

| Chapter | Title | Scope & Subsystems |
|---|---|---|
| **Intro** | [Introduction: The Discipline of Performance Engineering](00-introduction-the-discipline-of-performance-engineering.md) | System properties, USE/RED metrics, measurement imperative, cognitive traps |
| **01** | [The Performance Engineering Methodology](01-the-performance-engineering-methodology.md) | Workload characterization, SLI/SLO baselines, experimental controls, statistics |
| **02** | [Linux Kernel Architecture and Performance](02-linux-kernel-architecture-and-performance.md) | Boundary costs, system calls, tasks, context switching, softirqs, lock contention |
| **03** | [CPU Architecture and Performance Counters](03-cpu-architecture-and-performance-counters.md) | Superscalar pipelines, cache hierarchies, branch prediction, SMT, PMU counters |
| **04** | [CPU Scheduling and Process Placement](04-cpu-scheduling-and-process-placement.md) | CFS/EEVDF mechanics, virtual runtime, RT scheduling, affinity, NUMA balancing |
| **05** | [Memory Management: Virtual Memory and Pages](05-memory-management-virtual-memory-and-pages.md) | Page tables, minor/major faults, Buddy allocator, swap/zswap, Huge Pages, RSS/PSS |
| **06** | [NUMA Architecture and Memory Placement](06-numa-architecture-and-memory-placement.md) | Interconnect latency, local vs remote RAM, numactl policies, perf c2c, vNUMA |
| **07** | [The Page Cache and Memory-Mapped Files](07-the-page-cache-and-memory-mapped-files.md) | Folios, read-ahead, write-behind, mmap vs read, kswapd eviction, dirty writeback |
| **08** | [I/O Subsystems: From Block Layer to Disk](08-io-subsystems-from-block-layer-to-disk.md) | VFS-to-disk path, blk-mq, I/O schedulers, bio splitting/merging, queue depths |
| **09** | [Storage Technologies and Performance Characteristics](09-storage-technologies-and-performance-characteristics.md) | HDD seek physics, NAND/SSD wear leveling, NVMe queue pairs, RAID, cloud EBS |
| **10** | [Filesystems and Mount Options](10-filesystems-and-mount-options.md) | ext4 vs XFS vs btrfs, journaling modes, fsync cost, noatime, barrier tuning |
| **11** | [The Networking Stack and Performance](11-the-networking-stack-and-performance.md) | Socket buffers, NAPI softirqs, RSS/RPS/XPS, TCP window scaling, UDP overhead |
| **12** | [Network Performance Tuning in Practice](12-network-performance-tuning-in-practice.md) | TCP sysctls, BDP calculation, CUBIC vs BBR, multiqueue NICs, validation |
| **13** | [System Call Tracing and Process Analysis](13-system-call-tracing-and-process-analysis.md) | strace ptrace cost, perf trace, ftrace function_graph, dynamic kprobes/uprobes |
| **14** | [eBPF and BPF-Based Observability](14-ebpf-and-bpf-based-observability.md) | In-kernel VM, verifier, BPF maps, bpftrace scripts, BCC tools, CO-RE/libbpf |
| **15** | [Performance Analysis Tools and Flame Graphs](15-performance-analysis-tools-and-flame-graphs.md) | On-CPU vs Off-CPU flame graphs, perf record/report DWARF, hotspot call paths |
| **16** | [System Monitoring and Observability](16-system-monitoring-and-observability.md) | Classic tools (vmstat/iostat), modern tools, Pressure Stall Information (PSI) |
| **17** | [Benchmarking: Design and Execution](17-benchmarking-design-and-execution.md) | Workload synthesis, testbed isolation, warmup/cooldown, statistical confidence |
| **18** | [Benchmarking Tools and Suites](18-benchmarking-tools-and-suites.md) | fio storage jobs, stress-ng, sysbench OLTP, iperf3, lmbench, wrk, Phoronix |
| **19** | [CPU and Memory Tuning for Production](19-cpu-and-memory-tuning-for-production.md) | CPU governors, IRQ affinity, isolcpus/nohz_full, 1GB hugepages, vm sysctl knobs |
| **20** | [I/O and Filesystem Tuning for Production](20-io-and-filesystem-tuning-for-production.md) | Scheduler selection, mount flags, dirty page thresholds, NVMe queue polling |
| **21** | [cgroups, systemd and Resource Management](21-cgroups-systemd-and-resource-management.md) | cgroups v2 hierarchy, cpu.max quota throttling, memory.high/max, io limits |
| **22** | [Containers, Virtualization and Noisy Neighbors](22-containers-virtualization-and-noisy-neighbors.md) | Container vs VM overhead, hypervisor exits, cache thrashing, Intel RDT/MBA |
| **23** | [Databases and High-Performance Applications](23-databases-and-high-performance-applications.md) | PostgreSQL/MySQL memory & I/O patterns, shared buffers, futexes, runtime GC |
| **24** | [Distributed Systems and Cluster Performance](24-distributed-systems-and-cluster-performance.md) | Network serialization, PTP clock skew, Raft/Paxos disk fsync, cluster topology |
| **25** | [Production Case Studies](25-production-case-studies.md) | Real-world post-mortems: interrupt storms, memory swap crisis, NUMA skew, EBS stall |
| **26** | [Capacity Planning and Performance Regression](26-capacity-planning-and-performance-regression.md) | Little's Law, queueing models (M/M/c), baseline tracking, CI performance gates |
| **27** | [Advanced Topics and Emerging Technologies](27-advanced-topics-and-emerging-technologies.md) | PREEMPT_RT, kernel bypass (DPDK/XDP), x86 vs ARM64, RDMA, CXL interconnects |
| **28** | [Conclusion: Putting It All Together](28-conclusion-putting-it-all-together.md) | The engineer's mindset, Amdahl's Law, premature optimization, performance culture |
