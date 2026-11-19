---
title: 'Pre-Change Verification and Streaming Telemetry'
---

# Pre-Change Verification and Streaming Telemetry

Rendering configs from a Source of Truth stops typos, but it does not prove that a change is *safe*. A clean Jinja2 render can still introduce a shadowed ACL rule, drop a BGP session through a mismatched ASN, or blackhole a subnet. To catch design bugs *before* deployment, modern NetOps shifts verification left using static analysis (**Batfish**) and structured state snapshots (**Suzieq**). Once a change is live, polling SNMP every five minutes hides microbursts and rapid route flaps; **streaming telemetry** over gNMI pushes sub-second interface, BGP, and drop counters directly into Prometheus and Grafana so regressions surface immediately.

## Learning goals

By the end of this chapter you can:

- Use Batfish to verify reachability, detect shadowed ACL rules, and validate BGP session compatibility before touching a live router
- Capture and compare multi-vendor operational state snapshots with Suzieq before and after a maintenance window
- Explain how gNMI `SAMPLE` and `ON_CHANGE` subscriptions differ from SNMP polling
- Deploy a working streaming telemetry pipeline (`SR Linux` / `cEOS` gNMI → `gnmic` → `Prometheus` → `Grafana`)
- Write actionable PromQL alerts for BGP session drops, interface error spikes, and telemetry collector health

## Shifting left: pre-change verification

Testing network changes only in production is like deploying code without a compiler or unit tests. Pre-change verification splits into two complementary layers:

| Layer | Tooling | What it answers | When it runs |
|-------|---------|-----------------|--------------|
| Static config analysis | **Batfish** | "Given these candidate configs, will BGP form, are any ACL lines unreachable, and can subnet A reach subnet B?" | On every pull request, before deploy |
| Emulated lab smoke test | **Containerlab** | "Do the actual NOS binaries boot, converge, and pass packets?" | On pull request / staging |
| Operational state diff | **Suzieq** | "Did the live routing table, MAC table, EVPN routes, or BGP peer count change as expected?" | Immediately before and after live deploy |

Containerlab smoke testing is covered in [Containerlab in CI](07-containerlab-in-ci.md). Here we focus on Batfish for mathematical pre-change analysis and Suzieq for live state assertions.

## Static network analysis with Batfish

Batfish parses vendor configuration files (Cisco IOS/NX-OS, Arista EOS, Juniper Junos, FRR, Palo Alto, AWS/Azure security groups) into a vendor-neutral data-plane and control-plane model. Without booting a single router VM, it computes the exact FIBs, BGP ribs, and packet-forwarding behavior for every possible packet header.

### Snapshot directory layout

Batfish expects a snapshot directory containing a `configs/` folder (and optional `batfish/layer1_topology.json` for physical links):

```text
snapshot-candidate/
├── configs/
│   ├── spine01.cfg
│   ├── spine02.cfg
│   ├── leaf01.cfg
│   └── leaf02.cfg
└── batfish/
    └── layer1_topology.json
```

### Running pre-change assertions with pybatfish

Start the Batfish service (`docker run -d -p 9997:9997 -p 9996:9996 batfish/allinone`), then run a Python verification script in CI against your rendered candidate configs:

```python
#!/usr/bin/env python3
"""Pre-change verification gate using pybatfish."""
from pybatfish.client.session import Session
from pybatfish.datamodel import HeaderConstraints

bf = Session(host="127.0.0.1")
bf.set_network("dc-fabric")
bf.init_snapshot("snapshot-candidate", name="pr-42", overwrite=True)

# 1. Fail on any unparsed config lines or vendor syntax warnings
parse_warn = bf.q.initIssues().answer().frame()
assert parse_warn.empty, f"Config parse issues found:\n{parse_warn}"

# 2. Verify all configured BGP sessions are compatible (matching ASNs, IPs, hold timers)
bgp_compat = bf.q.bgpSessionCompatibility().answer().frame()
bad_bgp = bgp_compat[bgp_compat["Configured_Status"] != "UNIQUE_MATCH"]
assert bad_bgp.empty, f"Incompatible BGP sessions:\n{bad_bgp}"

# 3. Detect shadowed (unreachable) ACL / firewall filter lines
shadowed = bf.q.filterLineReachability().answer().frame()
assert shadowed.empty, f"Shadowed or unreachable ACL lines:\n{shadowed}"

# 4. Prove end-to-end reachability from leaf01 server VLAN to leaf02 server VLAN
reach = bf.q.reachability(
    pathConstraints={"startLocation": "leaf01[Vlan100]"},
    headers=HeaderConstraints(srcIps="10.20.1.0/24", dstIps="10.20.2.0/24", ipProtocols=["TCP"], dstPorts="443"),
    actions="FAILURE",
).answer().frame()
assert reach.empty, f"Found packets that fail to reach 10.20.2.0/24:443:\n{reach}"

print("All Batfish pre-change checks passed.")
```

> [!TIP]
> In the reachability query above, searching for `actions="FAILURE"` is far stronger than searching for `actions="SUCCESS"`. A `SUCCESS` query only proves *at least one* packet in the header space gets through; an empty `FAILURE` query mathematically proves **zero packets** in the entire `/24 → /24` range are dropped or looped.

## Operational state snapshots with Suzieq

Batfish analyzes *intended* configs; **Suzieq** collects *actual* live operational state (BGP, OSPF, EVPN, interfaces, routes, ARP/ND, MACs, LLDP) into a time-series Parquet store. That lets you run declarative assertions (`sq-poller` + `suzieq-cli`) before a change, execute the deployment, and diff the operational state afterward.

```bash
# Capture a pre-change baseline and verify BGP + interface health
suzieq-cli bgp assert --namespace=dc-fabric
suzieq-cli interface assert --namespace=dc-fabric

# Verify that total established BGP peers and IPv4 route counts match expectations
suzieq-cli bgp summarize --namespace=dc-fabric
suzieq-cli route count --namespace=dc-fabric --vrf=default

# After the change: diff routes added or withdrawn between pre-change and now
suzieq-cli route show --namespace=dc-fabric --start-time="10m ago" --view=changes
```

If an assertion fails—for example, a BGP session drops or an interface MTU mismatch appears—your CI/CD pipeline halts the rollout before touching the next rack (canary gating).

## Modern streaming telemetry with gNMI

Traditional SNMP polling (`GET` / `GETBULK` every 60–300 seconds) suffers from three structural flaws: it burns router CPU walking giant OID trees, it misses sub-minute microbursts and interface flaps between polls, and its MIBs lag years behind modern NOS features.

**gNMI (gRPC Network Management Interface)** reverses the model. Devices maintain a persistent HTTP/2 TLS stream and push YANG-modeled telemetry to a collector in two modes (see [YANG, NETCONF, and gNMI](06-yang-netconf-gnmi.md)):

- **`SAMPLE`**: pushes counters (interface octets, discards, optics DOM, CPU/memory) on a tight, deterministic interval (for example every 5–10 seconds).
- **`ON_CHANGE`**: pushes state transitions (BGP peer state changes, link up/down, transceiver insertion) **immediately** when the event occurs—zero polling delay and zero wasted bandwidth while steady.

```text
+--------------------+         gNMI (TCP 57400 / 6030, TLS)        +---------+
| Nokia SR Linux     | ──────────────────────────────────────────▶ |         |
| (OpenConfig / SRL) |   SAMPLE (10s counters) + ON_CHANGE (BGP)   |  gnmic  |
+--------------------+                                             |         |
                                                                   | (scrape |
+--------------------+         gNMI (TCP 6030, TLS)                |  :9804) |
| Arista cEOS / EOS  | ──────────────────────────────────────────▶ |         |
+--------------------+                                             +----+----+
                                                                        │
                                                           HTTP /metrics│
                                                                        ▼
                                                                 +------------+
                                                                 | Prometheus | ──▶ Alertmanager
                                                                 +-----+------+
                                                                       │
                                                                       ▼
                                                                 +------------+
                                                                 |  Grafana   |
                                                                 +------------+
```

## Building the pipeline: gnmic, Prometheus, and Grafana

`gnmic` is an open-source gNMI collector that subscribes to multi-vendor devices, normalizes paths and labels, and exposes a Prometheus `/metrics` endpoint (or writes directly to InfluxDB, Kafka, or VictoriaMetrics).

### 1. Collector configuration (`gnmic.yml`)

```yaml
# gnmic.yml
username: admin
password: ${GNMI_PASSWORD}
port: 57400
insecure: false
skip-verify: true # Lab only; use ca-file in production
encoding: json_ietf

targets:
  clab-fabric-spine01:
    address: clab-fabric-spine01:57400
    subscriptions: [srl-interfaces, srl-bgp]
  clab-fabric-leaf01:
    address: clab-fabric-leaf01:6030
    subscriptions: [eos-interfaces, eos-bgp]

subscriptions:
  srl-interfaces:
    paths:
      - /interface[name=ethernet-1/*]/statistics
      - /interface[name=ethernet-1/*]/oper-state
    stream-mode: sample
    sample-interval: 10s

  srl-bgp:
    paths:
      - /network-instance[name=default]/protocols/bgp/neighbor/session-state
      - /network-instance[name=default]/protocols/bgp/neighbor/afi-safi/received-routes
    stream-mode: on-change

  eos-interfaces:
    paths:
      - /interfaces/interface/state/counters
      - /interfaces/interface/state/oper-status
    stream-mode: sample
    sample-interval: 10s

  eos-bgp:
    paths:
      - /network-instances/network-instance[name=default]/protocols/protocol/bgp/neighbors/neighbor/state/session-state
    stream-mode: on-change

outputs:
  prom-out:
    type: prometheus
    listen: :9804
    path: /metrics
    metric-prefix: net
    append-subscription-name: false
    export-timestamps: true
    strings-as-labels: true
    expiration: 60s
```

> [!NOTE]
> Setting `strings-as-labels: true` turns string state values such as `session-state="established"` or `oper-state="up"` into Prometheus metric labels with value `1`, making state-transition alerts trivial to write in PromQL.

### 2. Prometheus scrape configuration (`prometheus.yml`)

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - alerts/network.yml

scrape_configs:
  - job_name: gnmic
    static_configs:
      - targets: ["gnmic:9804"]
```

### 3. Actionable PromQL alerts (`alerts/network.yml`)

Good network alerts page on **symptoms and sustained state loss**, not transient single-sample blips. Always include a meta-alert for the collector itself so a silent telemetry outage does not masquerade as a healthy network.

```yaml
groups:
  - name: network-telemetry
    rules:
      - alert: BgpPeerNotEstablished
        expr: |
          net_network_instance_protocols_bgp_neighbor_session_state{session_state!="established"} == 1
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "BGP peer {{ $labels.neighbor_peer_address }} down on {{ $labels.source }}"
          description: "Session state has been {{ $labels.session_state }} for >30s."

      - alert: InterfaceDiscardSpike
        expr: |
          (
            rate(net_interface_statistics_in_discarded_packets[2m])
            + rate(net_interface_statistics_out_discarded_packets[2m])
          ) > 50
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Sustained packet discards on {{ $labels.source }} {{ $labels.interface_name }}"
          description: "Discard rate is {{ $value | humanize }} pps over the last 2m."

      - alert: InterfaceFcsOrErrorRate
        expr: |
          rate(net_interface_statistics_in_error_packets[5m]) > 1
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "Physical layer errors on {{ $labels.source }} {{ $labels.interface_name }}"
          description: "Check optics DOM power and fiber patch cables."

      - alert: TelemetryTargetMissing
        expr: |
          absent(net_interface_statistics_in_octets{source="clab-fabric-spine01:57400"})
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "No gNMI telemetry from spine01 for >1m"
          description: "Verify gNMI daemon on spine01 and gnmic collector connectivity."
```

## Cardinality and operational guardrails

Streaming telemetry makes it easy to accidentally overwhelm Prometheus with high-cardinality time series:

- **Do not stream full BGP tables into Prometheus.** Streaming 900,000 Internet prefixes as Prometheus labels will OOM your TSDB in seconds. Export aggregate counters (`received-routes`, `active-routes`, `session-state`) to Prometheus, and send full RIB/BMP streams to a dedicated routing collector (see [BMP: BGP Monitoring Protocol](01c-bmp-bgp-monitoring.md)).
- **Scope wildcard paths deliberately.** Subscribing to `/` (the entire YANG tree) hammers the NOS management plane. Subscribe only to the specific subtrees (`statistics`, `oper-state`, `session-state`, optics DOM) you actually graph or alert on.
- **Pair telemetry with deployment annotations.** Emit a Grafana annotation whenever your Nornir/CI pipeline commits a change so every spike in discards or route churn immediately lines up with the exact commit that preceded it.

## Related chapters

- [Observability for Networks](01-observability-for-networks.md)
- [BMP: BGP Monitoring Protocol](01c-bmp-bgp-monitoring.md)
- [YANG, NETCONF, and gNMI](06-yang-netconf-gnmi.md)
- [Containerlab in CI: Smoke-Testing a Topology on Every Change](07-containerlab-in-ci.md)
- [Source of Truth, Config Templating, and Drift Detection](08-source-of-truth-templating-and-drift.md)
