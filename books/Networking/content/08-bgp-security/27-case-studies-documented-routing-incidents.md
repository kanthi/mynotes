---
title: 'Case Studies: Documented Routing Incidents'
---

# Case Studies: Documented Routing Incidents

Historical BGP incidents expose how protocol assumptions, automation bugs, and missing border filters combine to disrupt global connectivity. Studying the exact control-plane anatomy of each outage clarifies why no single defense—neither RPKI ROV nor prefix filtering alone—is sufficient on its own.

## 1. Pakistan Telecom / YouTube Hijack (February 24, 2008)

The 2008 YouTube outage remains the canonical example of an accidental **more-specific origin hijack** amplified by Longest Prefix Match (LPM).

### Incident Timeline (UTC)

- **18:47 UTC**: Acting on a domestic censorship directive from the Pakistan Telecommunication Authority, Pakistan Telecom (`AS17557`) injects a null-routed `208.65.153.0/24` into its internal routing domain and accidentally redistributes it to its international upstream, PCCW Global (`AS3491`).
- **18:48 UTC**: PCCW Global accepts the `/24` without an inbound customer prefix-list and propagates it across the global DFZ. Because YouTube (`AS36561`) only advertised the covering `/22` (`208.65.152.0/22`), LPM immediately steers global YouTube traffic into Pakistan Telecom's links, saturating them completely.
- **20:07 UTC**: YouTube responds by disaggregating its `/22` and announcing matching `/24` prefixes (`208.65.153.0/24`) to reclaim traffic on shorter paths.
- **20:51 UTC**: YouTube further announces `/25` prefixes (most of which are filtered by DFZ `/24` boundary policies, though some peers accept them).
- **21:01 UTC**: PCCW Global withdraws Pakistan Telecom's announcement and applies a filter, restoring global YouTube reachability after roughly two hours.

### Control-Plane Anatomy & Blast Radius

```text
Legitimate route:  208.65.152.0/22   AS_PATH: [..., 36561]      (YouTube)
Hijacked route:    208.65.153.0/24   AS_PATH: [..., 3491, 17557] (Pakistan Telecom)
Forwarding winner: 208.65.153.0/24   (Longest Prefix Match beats /22 globally)
```

- **Blast Radius**: Global blackhole of YouTube for ~2 hours; Pakistan Telecom's international transit links were completely overwhelmed by global video requests.
- **Controls That Would Have Prevented It**:
  - **RPKI ROV + Minimal ROA**: A ROA `(208.65.152.0/22, maxLength=22, AS36561)` would have marked `AS17557`'s `/24` as `Invalid` at PCCW Global and every ROV-enforcing peer.
  - **Strict IRR Customer Prefix Filtering**: PCCW Global filtering `AS17557` against its authorized IRR prefix-list would have blocked the announcement at the very first eBGP hop.

## 2. Verizon / Allegheny / Cloudflare Route Leak (June 24, 2019)

On June 24, 2019, a small Pennsylvania ISP caused a major US Internet outage affecting Cloudflare, AWS, and thousands of enterprise networks through a **Type-1 customer route leak amplified by a BGP route optimizer**.

### Incident Timeline (UTC)

- **10:35 UTC**: DQE Communications (`AS33154`) operates a commercial BGP route optimizer that dynamically injects more-specific `/21`–`/24` routes internally to steer outbound traffic. Due to a misconfiguration, these optimized more-specific routes leak to its customer, Allegheny Technologies (`AS396531`).
- **10:36 UTC**: Allegheny Technologies (`AS396531`) re-advertises those leaked more-specific routes to its upstream transit provider, **Verizon (`AS701`)**.
- **10:37 UTC**: Verizon (`AS701`) lacks inbound customer prefix filters and `maximum-prefix` caps on the `AS396531` session. Verizon assigns high customer `LOCAL_PREF` to the leaked routes and propagates them to its global Tier-1 peers and customers.
- **10:40–12:52 UTC**: Major networks—including Cloudflare (`AS13335`), whose legitimate aggregates were overridden by the optimizer's leaked more-specific `/24`s—suffer severe packet loss (~15% of Cloudflare's global traffic dropped) as traffic funnels into Allegheny's small edge circuit.
- **12:52 UTC**: Verizon filters and shuts down the leaking customer session after more than two hours of outage.

### Control-Plane Anatomy & Blast Radius

```text
Legitimate route:  104.16.0.0/12     AS_PATH: [..., 13335]               (Cloudflare)
Leaked route:      104.16.x.0/24     AS_PATH: [..., 701, 396531, 33154, 13335]
ROV status:        Valid / NotFound  (Origin AS13335 was NOT forged!)
```

- **Blast Radius**: Global disruption to Cloudflare, AWS, Linode, and regional US ISPs for over 2 hours.
- **Controls That Would Have Prevented It**:
  - **Strict Customer IRR Filters + `maximum-prefix`**: `AS396531` was a small enterprise stub; Verizon should never have accepted tens of thousands of third-party prefixes from it.
  - **RFC 9234 BGP Roles & OTC**: DQE (`AS33154`) would have stamped `OTC=33154` on egress to customer `AS396531`, and Verizon (`AS701`) would have automatically rejected any route carrying an `OTC` attribute on ingress from customer `AS396531`.
  - **ASPA**: Verizon would have detected the valley transition (`33154 -> 396531 -> 701`) and dropped the leaked paths.

> [!IMPORTANT]
> Notice why RPKI ROV did **not** stop the Verizon/Allegheny incident: for every prefix where the optimizer's more-specific fell within a loose `maxLength` (or had no ROA), the origin ASN (`AS13335`) remained legitimate.

## 3. Facebook / Meta Global Outage (October 4, 2021)

Unlike hijacks or external leaks, the October 4, 2021 Meta outage demonstrated the catastrophic impact of **self-inflicted BGP withdrawal combined with DNS/BGP fate-sharing**.

### Incident Timeline (UTC)

- **15:39 UTC**: During routine backbone capacity maintenance, an engineer issues a command intended to assess global backbone availability. A bug in Meta's internal audit tool fails to block the command, disconnecting Meta's backbone routers connecting its global data centers.
- **15:40 UTC**: Meta's authoritative DNS servers are designed to withdraw their BGP prefixes (`AS32934`) automatically whenever they lose connectivity to internal data centers (to avoid blackholing queries). Within seconds, **all** BGP routes to Meta's authoritative DNS resolvers (`a.ns.facebook.com`, etc.) are withdrawn from the global DFZ.
- **15:41–21:00 UTC**: Recursive resolvers worldwide receive `SERVFAIL` for `facebook.com`, `instagram.com`, and `whatsapp.com`. Worse, Meta's internal employee badge systems, out-of-band tooling, and remote management interfaces depended on the same internal DNS and backbone infrastructure, delaying physical recovery at data centers for nearly six hours.
- **21:09 UTC**: Engineers physically access core routers in Santa Clara, restore backbone state, and re-advertise `AS32934` prefixes; DNS caches recover globally over the next hour.

### Control-Plane Anatomy & Blast Radius

```text
15:39 UTC: Backbone disconnect -> Health-check fails on Edge PoP DNS servers
15:40 UTC: BGP WITHDRAW for all AS32934 authoritative DNS & edge prefixes
Result:    0 global paths to AS32934; global DNS resolution fails with SERVFAIL
```

- **Blast Radius**: ~6-hour global outage affecting ~3.5 billion users across Facebook, Instagram, WhatsApp, and Oculus, plus secondary retry-storm traffic spikes on public DNS resolvers (`1.1.1.1`, `8.8.8.8`).
- **Controls That Would Have Prevented It**:
  - **Independent Out-of-Band (OOB) Control Plane**: Never share DNS, authentication, or physical access control with the production BGP/backbone infrastructure being maintained.
  - **Canary & Blast-Radius Guardrails in Automation**: Enforce hard circuit-breakers in network automation (for example, refusing to withdraw >20% of global Anycast prefixes simultaneously without manual multi-party override).
  - **Secondary External DNS Diversity**: Hosting at least one authoritative DNS namespace on an independent network prevents total domain disappearance during an AS-wide withdrawal.

## 4. Rostelecom Financial & CDN Hijack Series (2017–2020)

Between 2017 and 2020, Russia's state-owned telecom provider **Rostelecom (`AS12389`)** was involved in multiple high-profile, short-duration BGP hijacks targeting financial institutions (Visa, Mastercard, Symantec in April 2017) and major CDNs and cloud providers (Google, Amazon, Cloudflare, Akamai—over 8,800 prefixes in April 2020).

### Incident Pattern & Control-Plane Anatomy

- **April 2017 (Financial Services)**: Rostelecom originated dozens of `/24` prefixes belonging to Visa, Mastercard, and financial infrastructure for roughly 7 minutes. Because many targets announced shorter aggregates (`/22`, `/20`), LPM diverted global traffic through `AS12389`.
- **April 1, 2020 (CDN & Cloud Event)**: For ~45 minutes, `AS12389` originated more than 8,800 more-specific prefixes belonging to over 200 major networks (including Google, AWS, Cloudflare, and DigitalOcean).

```text
Legitimate route:  104.18.0.0/20   AS_PATH: [..., 13335]   (Cloudflare)
Unauthorized:      104.18.x.0/24   AS_PATH: [..., 12389]   (Rostelecom)
ROV-enabled peers: DROPPED (Invalid - Origin mismatch)
Non-ROV peers:     ACCEPTED (Traffic diverted via AS12389)
```

- **Blast Radius**: In 2020, networks that had already deployed RPKI ROV (such as AT&T, Cloudflare, and major European IXP route servers) immediately dropped the unauthorized `AS12389` routes. Networks that had **not** deployed ROV accepted them, experiencing significant latency spikes and packet loss across Russia-detoured paths.
- **Controls That Would Have Prevented It**:
  - **Universal RPKI ROV Enforcement**: Directly neutralized the 2020 event for every adopting network.
  - **Minimal ROAs (RFC 9319)**: Prevented unauthorized `/24` sub-prefixes from slipping under loose `maxLength` thresholds.
  - **Real-Time BGP Monitoring (BGPalerter / BMP)**: Essential for catching short-duration (5–10 minute) hijacks like the 2017 financial event before data exfiltration or TLS interception succeeds.

## Comparative Incident and Control Matrix

| Incident | Year | Fault / Attack Class | Stopped by RPKI ROV? | Decisive Preventive Controls |
|:---|:---:|:---|:---:|:---|
| **Pakistan Telecom / YouTube** | 2008 | More-specific origin hijack (`/24` vs `/22`) | **YES** (Drop Invalid) | RPKI ROV + minimal ROA + upstream IRR customer filter |
| **Verizon / Allegheny / Cloudflare** | 2019 | Type-1 customer route leak + optimizer `/24`s | **NO** (Origin valid) | RFC 9234 BGP Roles/OTC + ASPA + customer IRR & `maximum-prefix` |
| **Facebook / Meta** | 2021 | Self-inflicted BGP withdrawal & fate-sharing | **N/A** (Legit withdraw) | Automation blast-radius guardrails + independent OOB management & DNS |
| **Rostelecom / Financial & CDNs** | 2017–2020 | Unauthorized more-specific origin announcements | **YES** (For ROV adopters) | Universal ROV enforcement + minimal ROAs + real-time BMP/BGPalerter |

> [!TIP]
> When reviewing any routing incident post-mortem, always ask two questions: (1) *Was the origin ASN altered?* (If yes, ROV + minimal ROAs stop it.) (2) *Was a legitimate route propagated across an illegal commercial boundary?* (If yes, RFC 9234 OTC, ASPA, and strict customer IRR/`maximum-prefix` filters are required.)
