---
title: '102i. Project: eBPF/XDP L4 Packet Filter & Container Syscall Tracer'
---

# Project: eBPF/XDP L4 Packet Filter & Container Syscall Tracer

Use **`cilium/ebpf`** in Go + C eBPF programs to build a dual-purpose kernel observability and data-plane daemon:
1. **XDP / `tc` L4 Packet Filter:** Drops blocked IPs and enforces per-IP token-bucket rate limits at the NIC driver level with lock-free `BPF_MAP_TYPE_PERCPU_ARRAY` counters.
2. **Container Syscall & Latency Tracer:** Attaches to `tracepoint/syscalls/sys_enter_execve`, `kprobe/tcp_v4_connect`, and `tracepoint/block/block_rq_issue` to surface noisy-neighbor disk/network stalls as Prometheus/OpenTelemetry histograms.

## Architecture

```text
Kernel Space (eBPF)                               User Space (Go + cilium/ebpf)
  NIC -> XDP Hook -> Check blocklist BPF map        ├── Loads & pins eBPF programs (bpf2go)
                  -> Update per-CPU drop counter    ├── Updates blocklist map via HTTP/CLI
  Tracepoints (execve / tcp_connect / blk_mq)  ---> ├── Reads RingBuf events (zero-copy)
                  -> Emit to BPF_MAP_TYPE_RINGBUF   └── Exports Prometheus /metrics & OTel
```

## Minimal XDP IPv4 Drop Filter (C)

```c
//go:build ignore
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 10240);
    __type(key, __u32);   // IPv4 source address
    __type(value, __u64); // drop count
} blocklist SEC(".maps");

SEC("xdp")
int xdp_filter(struct xdp_md *ctx) {
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end) return XDP_PASS;
    if (eth->h_proto != __constant_htons(ETH_P_IP)) return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end) return XDP_PASS;

    __u32 src = ip->saddr;
    __u64 *drops = bpf_map_lookup_elem(&blocklist, &src);
    if (drops) {
        __sync_fetch_and_add(drops, 1);
        return XDP_DROP;
    }
    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

## Acceptance Drills

- Generate Go bindings with `go generate ./...` (`bpf2go`), attach to a `veth` interface in a test network namespace, add a test IP to `blocklist`, and verify `XDP_DROP` increments the counter while non-blocked traffic passes at line rate.
- Trigger `fio` inside a container and verify `blk_mq` latency spikes appear on `/metrics`.
