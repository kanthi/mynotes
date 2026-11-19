---
title: 'Capstone C4: Five-Day Incident Week'
---

# Capstone C4: Five-Day Incident Week

Building a clean lab proves you can follow a blueprint; **recovering a broken lab under time pressure without guessing** proves you understand how networks actually fail. This chapter is the worked implementation of **Capstone C4** from [Capstone Outline](03-capstone-outline.md): a reusable Containerlab topology, five reproducible fault-injection scripts (one per day), structured **detect → diagnose → fix → prevent** runbooks, and an incident postmortem template.

## Learning goals

By the end of this chapter you can:

- Run a disciplined five-day diagnostic drill across Layer 2, static routing, MTU/ICMP, OSPF, and BGP
- Separate symptoms from root causes using evidence (`ip`, `bridge`, `vtysh`, `tcpdump`, `ping -M do`)
- Avoid "random change storms" by forming a falsifiable hypothesis before touching any config
- Convert every resolved incident into a permanent automated regression check in `verify.sh`
- Write concise, blameless incident notes and postmortems

## Reusable incident lab topology

This single 6-node lab exercises VLANs, OSPF, BGP, and MTU paths all at once so you can reuse the exact same `topology.clab.yml` for all five days:

```text
h1 (VLAN 10: 10.10.10.10/24)
       |
     [sw1] (Linux VLAN-aware bridge)
       | (802.1Q trunk: VLAN 10)
     [r1] (FRR: OSPF area 0 + VLAN 10 GW 10.10.10.1/24)
      / \
  eth2   eth3   (dual OSPF paths to r2; eth2 cost 10, eth3 cost 20)
      \ /
     [r2] (FRR: OSPF area 0 + eBGP AS 65001)
       | (eBGP to AS 65099)
     [isp] (FRR: AS 65099, lo 198.51.100.1/32, advertises 198.51.100.0/24)
       |
     [h2] (198.51.100.10/24 via 198.51.100.1)
```

### `topology.clab.yml`

```yaml
name: incweek

topology:
  nodes:
    h1:
      kind: linux
      image: alpine:3.20
      exec:
        - apk add --no-cache iproute2 iputils tcpdump
        - ip addr add 10.10.10.10/24 dev eth1
        - ip link set eth1 up
        - ip route add default via 10.10.10.1
    sw1:
      kind: linux
      image: alpine:3.20
      exec:
        - apk add --no-cache iproute2
        - ip link add br0 type bridge vlan_filtering 1
        - ip link set br0 up
        - ip link set eth1 master br0
        - ip link set eth2 master br0
        - ip link set eth1 up
        - ip link set eth2 up
        - bridge vlan del dev eth1 vid 1
        - bridge vlan del dev eth2 vid 1
        - bridge vlan add dev eth1 vid 10 pvid untagged
        - bridge vlan add dev eth2 vid 10
    r1:
      kind: linux
      image: quay.io/frrouting/frr:10.2.1
      binds:
        - ./config/r1:/etc/frr
      exec:
        - ip link add link eth1 name eth1.10 type vlan id 10
        - ip addr add 10.10.10.1/24 dev eth1.10
        - ip link set eth1 up
        - ip link set eth1.10 up
    r2:
      kind: linux
      image: quay.io/frrouting/frr:10.2.1
      binds:
        - ./config/r2:/etc/frr
    isp:
      kind: linux
      image: quay.io/frrouting/frr:10.2.1
      binds:
        - ./config/isp:/etc/frr
    h2:
      kind: linux
      image: alpine:3.20
      exec:
        - apk add --no-cache iproute2 iputils tcpdump
        - ip addr add 198.51.100.10/24 dev eth1
        - ip link set eth1 up
        - ip route add default via 198.51.100.1

  links:
    - endpoints: ["h1:eth1", "sw1:eth1"]
    - endpoints: ["sw1:eth2", "r1:eth1"]
    - endpoints: ["r1:eth2", "r2:eth1"]
    - endpoints: ["r1:eth3", "r2:eth2"]
    - endpoints: ["r2:eth3", "isp:eth1"]
    - endpoints: ["isp:eth2", "h2:eth1"]
```

### Baseline addressing and routing summary

- **r1–r2 primary (`eth2`–`eth1`):** `172.16.12.0/30` (OSPF area 0, cost 10)
- **r1–r2 backup (`eth3`–`eth2`):** `172.16.13.0/30` (OSPF area 0, cost 20)
- **r2–isp (`eth3`–`eth1`):** `172.16.29.0/30` (eBGP: `r2` AS `65001` ↔ `isp` AS `65099`)
- **r2** advertises `10.10.10.0/24` to `isp` via prefix-list `EXPORT-SITE` and redistributes/originates default (`default-information originate`) into OSPF toward `r1`.
- **Baseline `verify.sh`:** Checks `h1` → `10.10.10.1` (GW), `h1` → `198.51.100.10` (end-to-end), `h1` DF-bit 1472-byte ping (`1500` MTU), OSPF `Full` on `r1`/`r2`, and BGP `Established` on `r2`/`isp`.

> [!TIP]
> **How to run Incident Week solo:** Put all five `inject-dayN.sh` scripts in `scripts/`, then run a wrapper that picks one at random (or run them in order on five consecutive days) **without reading the script output** until after you finish diagnosing.

---

## Day 1 — Wrong VLAN on an access port

### Fault injection (`scripts/inject-day1.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail
# Move h1's access port from VLAN 10 to VLAN 20 silently
docker exec clab-incweek-sw1 bridge vlan del dev eth1 vid 10
docker exec clab-incweek-sw1 bridge vlan add dev eth1 vid 20 pvid untagged
```

### Runbook: detect → diagnose → fix → prevent

1. **Detect:** `h1` cannot ping its own default gateway `10.10.10.1` (`Destination Host Unreachable`).
2. **Diagnose:**
   - On `h1`: `ip neigh show` shows `10.10.10.1 FAILED` or `INCOMPLETE` — L2 ARP is broken.
   - On `r1`: `tcpdump -ni eth1.10 arp` sees **zero** ARP requests arriving.
   - On `sw1`: `bridge vlan show` reveals `eth1` is `20 PVID Egress Untagged` while trunk `eth2` only carries `10`.
3. **Fix:** Restore `eth1` to `vid 10 pvid untagged` on `sw1` and confirm ARP resolves on `h1`.
4. **Prevent:** Add a `verify.sh` check asserting `bridge vlan show dev eth1` contains `10 PVID Egress Untagged`.

---

## Day 2 — Missing return route (asymmetric blackhole)

### Fault injection (`scripts/inject-day2.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail
# Drop the static/BGP return route to 10.10.10.0/24 on isp
docker exec clab-incweek-isp vtysh \
  -c "conf t" \
  -c "ip prefix-list DENY-SITE seq 5 deny 10.10.10.0/24" \
  -c "ip prefix-list DENY-SITE seq 10 permit 0.0.0.0/0 le 32" \
  -c "route-map IN-R2 permit 10" \
  -c " match ip address prefix-list DENY-SITE" \
  -c "router bgp 65099" \
  -c " address-family ipv4 unicast" \
  -c "  neighbor 172.16.29.1 route-map IN-R2 in" \
  -c "end"
docker exec clab-incweek-isp vtysh -c "clear bgp ipv4 unicast 172.16.29.1 soft in"
```

### Runbook: detect → diagnose → fix → prevent

1. **Detect:** `h1` pings `10.10.10.1` and `r2` fine, but `ping 198.51.100.10` times out completely.
2. **Diagnose:**
   - Capture on `h2`: `docker exec clab-incweek-h2 tcpdump -ni eth1 icmp`. You see **ICMP Echo Requests arriving** from `10.10.10.10`, and `h2` sending Echo Replies back to `isp` — proving the **forward path works**!
   - Capture on `isp:eth1`: zero ICMP replies exit toward `r2`.
   - Check `isp` RIB: `vtysh -c "show ip route 10.10.10.0/24"` → `% Network not in table`.
   - Check `vtysh -c "show bgp neighbors 172.16.29.1 received-routes"` vs installed routes to spot the inbound `DENY-SITE` filter.
3. **Fix:** Remove the `IN-R2` deny entry (or restore the allow-list for `10.10.10.0/24`) and soft-clear inbound.
4. **Prevent:** Add a bidirectional route assertion in `verify.sh` checking both `r1` (`198.51.100.0/24`) **and** `isp` (`10.10.10.0/24`).

---

## Day 3 — PMTU black hole + ICMP filter

### Fault injection (`scripts/inject-day3.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail
# Lower MTU on the r2<->isp transit link to 1300 AND drop ICMP Fragmentation Needed on r2
docker exec clab-incweek-r2 ip link set eth3 mtu 1300
docker exec clab-incweek-isp ip link set eth1 mtu 1300
docker exec clab-incweek-r2 iptables -I OUTPUT -p icmp --icmp-type fragmentation-needed -j DROP
```

### Runbook: detect → diagnose → fix → prevent

1. **Detect:** Small pings (`ping -c 3 198.51.100.10`) **succeed**, SSH connects, yet large data transfers or full-size packets hang silently!
2. **Diagnose:**
   - Test path MTU with the Don't Fragment (`DF`) bit set from `h1`:
     ```bash
     docker exec clab-incweek-h1 ping -c 2 -M do -s 1472 198.51.100.10  # FAILS (1500 B)
     docker exec clab-incweek-h1 ping -c 2 -M do -s 1272 198.51.100.10  # SUCCEEDS (1300 B)
     ```
   - Notice `h1` never receives `Frag needed and DF set` — a classic **PMTUD black hole**.
   - Inspect `r2` interface MTUs (`ip link show eth3` → `mtu 1300`) and firewall counters (`iptables -L OUTPUT -nv` → incrementing drops on `fragmentation-needed`).
3. **Fix:** Restore `mtu 1500` on `r2:eth3` and `isp:eth1`, and remove the ICMP Type 3 Code 4 `DROP` rule.
4. **Prevent:** Include `ping -c 2 -M do -s 1472 198.51.100.10` in `verify.sh` so MTU regressions fail CI immediately.

---

## Day 4 — OSPF cost surprise and link flap

### Fault injection (`scripts/inject-day4.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail
# Inflate OSPF cost on the primary link r1:eth2 so traffic shifts to backup eth3,
# and add 150ms latency + 10% packet loss on the backup link to simulate a degraded path
docker exec clab-incweek-r1 vtysh \
  -c "conf t" -c "interface eth2" -c "ip ospf cost 500" -c "end"
docker exec clab-incweek-r1 tc qdisc replace dev eth3 root netem delay 150ms loss 10%
```

### Runbook: detect → diagnose → fix → prevent

1. **Detect:** Reachability still works, but latency between `h1` and `h2` jumps from `<2 ms` to `~150 ms` with intermittent packet loss.
2. **Diagnose:**
   - `traceroute -n 198.51.100.10` from `h1` shows traffic taking `172.16.13.2` (backup link `eth3`) instead of `172.16.12.2` (primary `eth2`).
   - On `r1`: `vtysh -c "show ip ospf interface brief"` shows `eth2` is UP/Full, but its **Cost is 500** (higher than `eth3`'s cost of 20).
   - Inspect `tc qdisc show dev eth3` to confirm packet impairment on the backup path.
3. **Fix:** Restore `ip ospf cost 10` on `r1:eth2` and clear `tc qdisc del dev eth3 root`.
4. **Prevent:** Add a primary-path nexthop check (`ip route get 198.51.100.10` must use `172.16.12.2`) and an RTT threshold check to `verify.sh`.

---

## Day 5 — Overly strict BGP prefix-list

### Fault injection (`scripts/inject-day5.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail
# Change r2's outbound prefix-list to require /25 instead of /24 (prefix never matches!)
docker exec clab-incweek-r2 vtysh \
  -c "conf t" \
  -c "no ip prefix-list EXPORT-SITE" \
  -c "ip prefix-list EXPORT-SITE seq 10 permit 10.10.10.0/25" \
  -c "end"
docker exec clab-incweek-r2 vtysh -c "clear bgp ipv4 unicast 172.16.29.2 soft out"
```

### Runbook: detect → diagnose → fix → prevent

1. **Detect:** BGP session `r2 ↔ isp` is `Established`, yet `h1 → h2` fails completely.
2. **Diagnose:**
   - On `r2`: `vtysh -c "show bgp ipv4 unicast neighbors 172.16.29.2 advertised-routes"` returns **Total number of prefixes 0**!
   - Inspect `show ip prefix-list EXPORT-SITE` on `r2`: it permits `10.10.10.0/25` (exact `/25`), whereas the site subnet in BGP is `10.10.10.0/24`.
3. **Fix:** Restore `ip prefix-list EXPORT-SITE seq 10 permit 10.10.10.0/24` and soft-clear outbound.
4. **Prevent:** Add an advertised-prefix count check (`advertised-routes` must include `10.10.10.0/24`) to `verify.sh`.

---

## Incident note and postmortem template

Copy this template into `journals/day-N-incident.md` for every drill:

```markdown
# Incident Note — Day N: <Short Title>

- **Date / Duration:** YYYY-MM-DD | Detected: HH:MM | Mitigated: HH:MM (TTR: X min)
- **Severity / Impact:** (e.g., Total loss of h1 -> h2 reachability; latency degraded to 150 ms)

## 1. Symptoms Observed
- Exact failing commands and error outputs (paste verbatim).

## 2. Diagnostic Timeline & Hypotheses
- HH:MM — Hypothesis 1: <what we suspected> → Command run → Result (ruled out / confirmed).
- HH:MM — Hypothesis 2: ...

## 3. Root Cause
- Exact misconfiguration or failure, with the proving command output.

## 4. Fix & Verification
- Exact change applied.
- `./verify.sh` output after recovery.

## 5. Preventive Action (Added to `verify.sh`)
- Exact automated check added so this failure mode is caught immediately next time.
```

## Definition of done

- [ ] Reusable `topology.clab.yml`, baseline configs, and all five `inject-dayN.sh` scripts in git
- [ ] All 5 days completed with written incident notes in `journals/day-1.md` through `day-5.md`
- [ ] `verify.sh` expanded with **at least 5 new automated checks** (one per incident) and passes green
- [ ] `RETRO.md` summarizing which diagnostic commands gave the highest signal-to-noise ratio
