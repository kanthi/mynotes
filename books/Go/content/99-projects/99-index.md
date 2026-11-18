---
title: 000 Projects Index
---

# 000 Projects: Build Real Software

This section is hands-on. Most chapters are **build projects** (runnable code + steps). A few are **checklist / drill** projects (harden or verify an existing service against a list)—those are labeled in the chapter title (e.g. *Secure Service Checklist*).

## Project tiers

| Tier | What you get |
|------|----------------|
| **Build** | Module layout, full or nearly full source, run instructions, extensions |
| **Design-and-build lab** | Contracts, Go implementation milestones, failure drills, and acceptance evidence; you implement the complete service |
| **Checklist** | Requirements and evidence to collect; you supply or reuse a small service |
| **Index / ideas** | Catalogues and workout lists (not a single binary) |

## How To Use These Projects

1. Read the architecture diagram.
2. Type the code yourself first.
3. Run it.
4. Break it intentionally.
5. Add one extension before moving to the next project.

```text
Learning loop

read -> build -> run -> debug -> extend -> repeat
```

## Project Path (Beginner -> Advanced)

1. [CLI Ping Tool](99a-cli-ping-tool.md)
2. [URL Shortener API](99b-url-shortener-api.md)
3. [Concurrent Log Analyzer](99c-concurrent-log-analyzer.md)
4. [File Integrity Checker](99d-file-integrity-checker.md)
5. [Concurrent Port Scanner](99e-concurrent-port-scanner.md)
6. [Mini Job Runner](99f-mini-job-runner.md)
7. [Linux `ls` Clone](99g-linux-ls-clone.md)
8. [Linux `cat` Clone](99h-linux-cat-clone.md)
9. [Linux `grep` Clone](99i-linux-grep-clone.md)
10. [Linux `tail -f` Clone](99j-linux-tail-follow.md)
11. [Linux `du` Clone](99k-linux-du-clone.md)
12. [Linux `find` Clone](99l-linux-find-clone.md)
13. [Linux `wc` Clone](99m-linux-wc-clone.md)
14. [HTTP Load Tester](99n-http-load-tester.md)
15. [TUI System Monitor](99o-tui-system-monitor.md)
16. [Proxmox TUI Manager](99p-proxmox-tui-manager.md)
17. [SSH Remote Orchestrator](99q-ssh-remote-orchestrator.md)
18. [Go Workout: 200 Ten-Minute Exercises](99r-go-workout-200-exercises.md)
19. [Linux ps-lite](99s-linux-ps-lite.md)
20. [KV HTTP Store](99t-kv-http-store.md)
21. [WebSocket Chat](99u-websocket-chat.md)
22. [Log Shipper CLI](99v-log-shipper-cli.md)
23. [Proxmox Batch Operations CLI](99w-proxmox-batch-cli.md)
24. [Controller Pattern Simulator](99x-controller-pattern-sim.md)
25. [Resource Index + 150 More Ideas](99y-resource-index-and-ideas.md)
26. [Kubernetes Pod Lister](99z-k8s-pod-lister.md)
27. [Kubernetes Event Watcher](100a-k8s-event-watcher.md)
28. [Kubernetes Rollout Checker](100b-k8s-rollout-checker.md)
29. [Terraform Plan Parser](100c-terraform-plan-parser.md)
30. [Terraform Risk Reporter](100d-terraform-risk-reporter.md)
31. [Prometheus Exporter](100e-prometheus-exporter.md)
32. [Prometheus Probe Service](100f-prometheus-probe-service.md)
33. [eBPF Tracepoint Basics](100g-ebpf-tracepoint-basics.md)
34. [eBPF XDP Packet Counter](100h-ebpf-xdp-packet-counter.md)
35. [Proxmox Multi-Node Scheduler](100i-proxmox-multi-node-scheduler.md)
36. [Proxmox Capacity Balancer](100j-proxmox-capacity-balancer.md)
37. [Kubernetes HPA Simulator](100k-k8s-hpa-simulator.md)
38. [Terraform Cost Estimator](100l-terraform-cost-estimator.md)
39. [Prometheus Alert Rule Tester](100m-prometheus-alert-rule-tester.md)
40. [Infra Reconciler Daemon](100n-infra-reconciler-daemon.md)

### TinyGo & soft realtime (embedded)

41. [TinyGo Blink & Toolchain](100o-tinygo-blink-and-toolchain.md)
42. [TinyGo Button + Interrupt](100p-tinygo-button-interrupt.md)
43. [TinyGo UART Sensor Loop](100q-tinygo-uart-sensor-loop.md)
44. [TinyGo Cooperative Tasks (Soft RT)](100r-tinygo-cooperative-tasks.md)
45. [TinyGo I2C Realtime Monitor](100s-tinygo-i2c-realtime-monitor.md)

```text
TinyGo track

toolchain/blink -> button/IRQ -> timed sensor UART -> cooperative tasks -> I2C monitor
```

## System-design capstones

The [System Design in Go track](101-system-design/index.md) maps 28 design topics to this book. It extends the shortener, KV store, and chat builds with durability and multi-instance exercises, then adds 17 labs covering feeds, crawling, autocomplete, notifications, object storage, media, file sync, geospatial queries, routing, event aggregation, reservations, email, leaderboards, payments, wallets, and matching engines. Shared queue and monitoring labs live in Distributed Infrastructure and Observability.

## What You Will Practice

- CLI flags and argument parsing
- Networking and HTTP servers
- Goroutines, channels, and worker pools
- File I/O and hashing
- Error handling and retries
- Production-friendly project structure
- **TinyGo** targets, `machine` pins, soft-realtime loops, interrupts, UART/I2C


## Step-by-Step Explanation

1. Pick a small set of exercises by track.
2. Timebox each attempt to ten minutes.
3. Record one takeaway and one weakness after each exercise.
4. Revisit chapter references when blocked.
5. Re-solve selected problems from memory weekly.

## Learning Goals

- Build consistency, not one-time intensity.
- Improve retrieval and transfer of Go patterns.
- Progress from syntax fluency to engineering fluency.
