---
title: The Scope and Limits of RPKI and ROV
---

# The Scope and Limits of RPKI and ROV

RPKI Route Origin Validation (ROV, RFC 6811) is the single highest-impact cryptographic control deployed in interdomain routing today. However, treating ROV as a complete routing security architecture leads to dangerous blind spots. Architects must understand precisely what ROV proves, where its guarantees stop, and which complementary controls are required to close the remaining gaps.

## What RPKI ROV Proves vs. What It Does Not

ROV evaluates a received BGP route against a local table of Validated ROA Payloads (VRPs), each encoding `(prefix, maxLength, origin_ASN)`. Its cryptographic guarantee is strictly scoped to **origin authorization** and **prefix-length bounds**:

| Security Property | Proven by RPKI ROV? | Technical Reason |
|:---|:---:|:---|
| **Origin ASN Authorization** | **YES** | Confirms the rightmost ASN in `AS_PATH` is authorized by the IP resource holder's signed ROA. |
| **Prefix Length Bound** | **YES** | Confirms the announced mask length does not exceed the ROA's `maxLength`. |
| **AS_PATH Authenticity** | **NO** | ROV never inspects intermediate ASNs; a forged path `[Attacker, Victim]` evaluates as `Valid`. |
| **Route Leak Prevention** | **NO** | Leaked routes preserve the legitimate origin ASN and prefix length, so they evaluate as `Valid`. |
| **Data-Plane Integrity** | **NO** | ROV is purely control-plane; an on-path router with a `Valid` route can still drop, delay, or intercept packets. |
| **Unregistered Prefix Protection** | **NO** | Prefixes without a covering ROA evaluate as `NotFound`, which networks must accept by default. |

## The Three ROV Validation States and the `NotFound` Fallback

Every BGP prefix evaluated by ROV falls into one of three states defined in RFC 6811:

1. **`Valid`**: At least one VRP covers the prefix, the announced prefix length is less than or equal to `maxLength`, and the origin ASN matches the VRP's ASN.
2. **`Invalid`**: At least one VRP covers the prefix, but *no* covering VRP matches both the origin ASN and the `maxLength` constraint. ROV-enforcing routers drop `Invalid` routes at ingress.
3. **`NotFound` (Unknown)**: No VRP covers the announced prefix.

Because global ROA coverage is still incomplete, networks cannot drop `NotFound` routes without breaking reachability to a significant portion of the Internet. That operational reality creates a persistent gap:

- **Dormant or unprotected address space** without a ROA (or an `AS0` ROA explicitly forbidding origination) evaluates as `NotFound` and passes right through ROV filters.
- **Partial-coverage hijacks**: If a victim only creates ROAs for half of its allocated prefixes, attackers can still hijack the un-ROA'd blocks without triggering `Invalid`.

> [!TIP]
> For allocated address space that you do **not** currently announce in BGP, publish an **`AS0` ROA** (RFC 6483). Because AS 0 is reserved and cannot legitimately originate routes, any announcement covering that prefix evaluates as `Invalid` and is dropped by ROV-enforcing networks.

## Why ROV Cannot Stop Route Leaks or Forged-Origin Hijacks

Two major classes of routing incidents routinely pass ROV with a `Valid` result:

### Customer-to-Provider Route Leaks

Suppose `AS15169` originates `8.8.8.0/24` with a valid ROA `(8.8.8.0/24, maxLength=24, AS15169)`:

1. Transit Provider A advertises `8.8.8.0/24` (`AS_PATH: [Provider_A, 15169]`) to Customer B.
2. Customer B misconfigures its export filter and leaks the route to Transit Provider C (`AS_PATH: [Customer_B, Provider_A, 15169]`).
3. Provider C runs ROV: the prefix is `8.8.8.0/24`, the mask is `/24`, and the rightmost ASN is `15169`.
4. Result: **`Valid`**. Provider C installs the leaked route with high customer `LOCAL_PREF`, pulling global traffic into Customer B's congested edge.

### Forged-Origin (Type-1) Hijacks

An attacker (`AS64599`) simply appends the victim's ASN to its own announcement (`AS_PATH: [64599, 15169]`). Because ROV checks only the rightmost ASN, the forged route also evaluates as **`Valid`**.

## The `maxLength` Over-Permissiveness Trap

A common operational mistake is setting `maxLength` looser than the prefixes actually announced in BGP (for example, announcing `203.0.113.0/22` with a ROA specifying `maxLength=24`).

- If an attacker executes a forged-origin hijack for `203.0.113.0/24` (`AS_PATH: [Attacker, Victim]`), the `/24` satisfies `maxLength=24` and matches the victim origin.
- The hijacked `/24` is marked **`Valid`** and wins globally via Longest Prefix Match over the victim's legitimate `/22`.

RFC 9319 recommends **minimal ROAs**: never set `maxLength` longer than the most-specific prefix actually originated. If you announce both a `/22` and one specific `/24`, publish two exact-length ROAs (`/22 maxLength 22` and `/24 maxLength 24`) instead of a single `/22 maxLength 24`.

## RTR Cache and Relying Party Failure Modes

Deploying ROV couples your BGP control plane to an external cryptographic pipeline: RIR Trust Anchors $\to$ RRDP/rsync repositories $\to$ Relying Party (RP) validator $\to$ RPKI-to-Router (RTR, RFC 8210) sessions $\to$ border routers. Several failure modes must be engineered for:

| Failure Mode | Immediate Symptom | Operational Mitigation |
|:---|:---|:---|
| **Single RP Validator Crash** | Router loses RTR session; after the RTR `expire` timer (default ~1–2 hours), router flushes its VRP table and all routes revert to `NotFound` | Run at least two independent RP validators (ideally diverse implementations, e.g., `rpki-client` + Routinator) with redundant RTR sessions per border router |
| **Upstream CA / RRDP Outage** | Validator cannot fetch updated manifests; cached objects remain valid until `nextUpdate` expires, then disappear | Monitor VRP count deltas and RRDP HTTP error rates; alert before `expire` windows lapse |
| **NTP Clock Drift** | X.509 certificates and manifests fail `notBefore`/`notAfter` validation, collapsing the entire VRP cache | Enforce authenticated redundant NTP on all validator hosts and routers |
| **Stale Human ROA Data** | Operator migrates a prefix to a new origin ASN or DDoS scrubber without updating the ROA; legitimate traffic is dropped globally as `Invalid` | Automate ROA updates in CI/CD alongside BGP change requests; add new origin ROAs *before* announcing and remove old ROAs *after* withdrawing |

> [!WARNING]
> Never deploy a single RPKI validator in production. If a single RP instance fails or returns a corrupted cache, your border routers either lose ROV protection (fail-open to `NotFound`) or, worse, reject valid routes if a broken local override is pushed.

### Illustrative FRR RTR Redundancy Check

Operators verify redundant RTR sessions and VRP cache health directly on edge routers (output below is illustrative):

```bash
# Illustrative FRR command (not executed on a live host)
vtysh -c "show rpki cache-connection"
```

```text
# Illustrative output
Connected to group 1
  rpki tcp cache 192.0.2.10 3323 pref 1 (connected)
  rpki tcp cache 192.0.2.11 3323 pref 2 (connected)
Total IPv4 VRPs: 468,912
Total IPv6 VRPs: 112,405
```

## Defense-in-Depth: Layering ROV with Complementary Controls

Because ROV solves origin authorization and nothing else, production networks must layer five complementary mechanisms at every eBGP boundary:

| Control Layer | Primary Threat Mitigated | Where Applied | Covers ROV Gaps? |
|:---|:---|:---|:---|
| **RPKI ROV (RFC 6811)** | Accidental & malicious origin hijacks, unauthorized sub-prefixes | All eBGP ingress (drop `Invalid`) | Baseline cryptographic origin check |
| **IRR Prefix & AS-Path Filters** | Unauthorized customer announcements, `NotFound` hijacks, forged origins from customers | Customer & peer eBGP ingress (generated via `bgpq4`) | Restricts accepted prefixes to explicit customer cones |
| **`maximum-prefix` Limits** | Full-table route leaks from customers or peers exhausting router RIB/FIB memory | Every eBGP session (with warning + teardown thresholds) | Caps blast radius of accidental leaks |
| **RFC 9234 BGP Roles & OTC** | Valley route leaks (`Provider -> Customer -> Provider` and `Peer -> Peer`) | All eBGP sessions supporting Capability 9 | Automatically marks and drops leaked routes |
| **ASPA (IETF sidrops)** | Forged-origin hijacks and route leaks across multiple hops | Upstream, peer, and customer eBGP ingress | Cryptographically validates customer-to-provider path pairs |

### Common Misconceptions Summary

1. **"RPKI encrypts Internet traffic."** — False. RPKI signs control-plane routing assertions; it does not encrypt data-plane packets (that is the role of TLS, IPsec, or MACsec).
2. **"Enabling ROV means I can remove IRR prefix-lists."** — False. Until 100% of prefixes have ROAs and ASPA is universally deployed, `NotFound` routes and forged-origin hijacks still require strict IRR customer-cone filters.
3. **"ROV stops route leaks."** — False. Route leaks carry legitimate origin ASNs and evaluate as `Valid`; preventing them requires RFC 9234 OTC, `maximum-prefix`, and ASPA.

> [!NOTE]
> Treat RPKI ROV as the foundation of routing hygiene, not the ceiling. Combining ROV (drop `Invalid`), minimal ROAs (RFC 9319), automated `bgpq4` IRR filters, `maximum-prefix`, RFC 9234 OTC, and ASPA provides comprehensive protection across both origin and path dimensions.
