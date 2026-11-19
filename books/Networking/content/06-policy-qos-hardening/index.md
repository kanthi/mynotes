---
title: Intro
---

# Intro

**Policy, QoS, and Hardening** — Protect devices and traffic across control, management, and data planes: ACLs, CoPP, BGP session protection, AAA/802.1X, QoS/FQ-CoDel/CAKE/L4S, `nftables`, conntrack tuning, MACsec, RTBH/Flowspec, and IPv6 First-Hop Security.

## How to use this part

Use `04-hardening-checklists.md` as a baseline audit and dive into `05`–`08` for Linux firewall and queue tuning.

## Chapters in this part

| Chapter | File |
|---------|------|
| [ACL Thinking](01-acl-thinking.md) | `01-acl-thinking.md` |
| [Control-Plane Protection](02-control-plane-protection.md) | `02-control-plane-protection.md` |
| [Protecting the BGP Session: TCP-MD5, TCP-AO and GTSM on Linux](02b-bgp-session-protection-md5-ao-gtsm.md) | `02b-bgp-session-protection-md5-ao-gtsm.md` |
| [Device AAA with RADIUS and TACACS+: Shared Secrets, MD5 and BlastRADIUS](02c-device-aaa-radius-tacacs-blastradius.md) | `02c-device-aaa-radius-tacacs-blastradius.md` |
| [Network Access Control: 802.1X, EAP and Authorization](02d-network-access-8021x.md) | `02d-network-access-8021x.md` |
| [QoS Vocabulary](03-qos-vocabulary.md) | `03-qos-vocabulary.md` |
| [QoS: Queues, Token Buckets and Congestion Evidence](03b-qos-queues-and-token-buckets.md) | `03b-qos-queues-and-token-buckets.md` |
| [Hardening Checklists](04-hardening-checklists.md) | `04-hardening-checklists.md` |
| [SYN Floods and Listen Queues: SYN_RECV, SYN Cookies, and Accept Overflow](05-syn-floods-and-listen-queues.md) | `05-syn-floods-and-listen-queues.md` |
| [FQ-CoDel and CAKE: How Linux Fights Bufferbloat, and How to Check It Is Actually](06-fq-codel-and-cake.md) | `06-fq-codel-and-cake.md` |
| [ECN and L4S: Marking Instead of Dropping](06b-ecn-and-l4s.md) | `06b-ecn-and-l4s.md` |
| [nftables Fundamentals: Tables, Chains, Sets and NAT](07-nftables-fundamentals.md) | `07-nftables-fundamentals.md` |
| [Conntrack Exhaustion: Limits, Timeouts and the Table-Full Drop](08-conntrack-exhaustion.md) | `08-conntrack-exhaustion.md` |
| [Stateful Firewall HA and Conntrack Sync](08b-stateful-firewall-ha-and-conntrack-sync.md) | `08b-stateful-firewall-ha-and-conntrack-sync.md` |
| [MACsec (IEEE 802.1AE) and MKA: Hop-by-Hop Ethernet Encryption](09-macsec-and-mka.md) | `09-macsec-and-mka.md` |
| [Dropping Attack Traffic with Routing: RTBH, Source-Based RTBH and BGP FlowSpec](10-rtbh-and-flowspec.md) | `10-rtbh-and-flowspec.md` |
| [IPv6 First-Hop Security and Binding Validation](11-ipv6-first-hop-security.md) | `11-ipv6-first-hop-security.md` |
