---
title: '04. Project: NVMe/TCP Storage QoS & Kernel eBPF/ftrace Diagnostics Toolkit'
---

# Project: NVMe/TCP Storage QoS & Kernel eBPF/`ftrace` Diagnostics Toolkit

Combine two deep Linux performance and kernel troubleshooting labs:
1. **Linux NVMe/TCP Storage Performance & `cgroup v2` QoS Harness:** Serve NVMe-oF/TCP targets between two Linux nodes, run `fio` + PostgreSQL `pgbench`, and tune `blk-mq`, `io_uring`, WAL `fsync`, and `cgroup v2` `io.max` limits to eliminate noisy-neighbor tail latency.
2. **Kernel `strace`/`ftrace`/`perf`/`bpftrace` Diagnostics Toolkit:** Detect anomalous `execve` calls, file-descriptor leaks, TCP retransmits, and block I/O latency spikes in containerized workloads.

## Isolating Noisy Neighbors with `cgroup v2` `io.max`

```bash
# Cap a noisy batch container's NVMe device (major:minor 259:0) to 50 MB/s read/write & 2000 IOPS
echo "259:0 rbps=52428800 wbps=52428800 riops=2000 wiops=2000" \
  > /sys/fs/cgroup/noisy-batch.slice/io.max
```

## Quick `bpftrace` One-Liners for Container Triage

```bash
# Block I/O latency histogram (microseconds)
sudo bpftrace -e 'tracepoint:block:block_rq_issue { @start[args->dev, args->sector] = nsecs; }
  tracepoint:block:block_rq_complete /@start[args->dev, args->sector]/ {
    @us = hist((nsecs - @start[args->dev, args->sector]) / 1000);
    delete(@start[args->dev, args->sector]);
  }'

# Live TCP retransmits by process
sudo bpftrace -e 'tracepoint:tcp:tcp_retransmit_skb { printf("%s pid=%d\n", comm, pid); }'
```

## Acceptance Drills

- Run `pgbench` alongside an uncapped `fio` randwrite job, record p99 transaction latency, apply `io.max`, and verify p99 latency drops back near baseline.
