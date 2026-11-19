---
title: Intro
---

# Intro

**Network Services** — Operate the core infrastructure services that every network relies on: first-hop redundancy (VRRP), DHCP/NAT/PXE, NTP/PTP/NTS time sync, authoritative and recursive DNS, IPv6 transition mechanisms, and mDNS.

## How to use this part

This part is modular—use the table below to jump directly to DNS (`05`–`19`), DHCP/NAT (`02`–`02d`, `21`–`22`), or time/logging (`03`–`03c`).

## Chapters in this part

| Chapter | File |
|---------|------|
| [First-Hop Redundancy](01-first-hop-redundancy.md) | `01-first-hop-redundancy.md` |
| [DHCP Relay and NAT](02-dhcp-relay-and-nat.md) | `02-dhcp-relay-and-nat.md` |
| [Network Boot: PXE, iPXE, UEFI HTTP Boot and TFTP](02b-network-boot-pxe-ipxe-tftp.md) | `02b-network-boot-pxe-ipxe-tftp.md` |
| [DHCP Option 82: Relay Agent Information, Circuit IDs and the Renewal Gap](02c-dhcp-option-82-relay-agent-information.md) | `02c-dhcp-option-82-relay-agent-information.md` |
| [DHCPv4 Lease and Relay Troubleshooting](02d-dhcp-lease-relay-diagnostics.md) | `02d-dhcp-lease-relay-diagnostics.md` |
| [Name, Time, and Log Hygiene](03-name-time-log-hygiene.md) | `03-name-time-log-hygiene.md` |
| [Precision Time Protocol (PTP): Beyond NTP](03b-ptp-precision-time.md) | `03b-ptp-precision-time.md` |
| [Network Time Security: NTS-KE, Cookies and Authenticated NTP (RFC 8915)](03c-nts-for-ntp.md) | `03c-nts-for-ntp.md` |
| [DNS Authoritative and Recursive](05-dns-authoritative-and-recursive.md) | `05-dns-authoritative-and-recursive.md` |
| [The Linux Stub Resolver Measured: resolv.conf, Search Lists, ndots, Timeouts](05b-linux-stub-resolver.md) | `05b-linux-stub-resolver.md` |
| [Zone Transfers: AXFR, IXFR, NOTIFY and TSIG](05c-zone-transfers-notify-tsig.md) | `05c-zone-transfers-notify-tsig.md` |
| [DNS Rebinding: How a Web Page Reaches Your LAN, and the Three Places to Stop](05d-dns-rebinding-defenses.md) | `05d-dns-rebinding-defenses.md` |
| [Negative Answers in DNS: NXDOMAIN, NODATA, Negative Caching and Compact Denial](05e-dns-negative-answers-and-caching.md) | `05e-dns-negative-answers-and-caching.md` |
| [EDNS Client Subnet: Geo-Answers, Cache Scope, Privacy and Split-Horizon Pitfalls](05f-edns-client-subnet-and-split-horizon.md) | `05f-edns-client-subnet-and-split-horizon.md` |
| [DNS Cache Poisoning After Kaminsky: TXID, 0x20-Bit and Cookies](05g-dns-cache-poisoning-0x20-cookies.md) | `05g-dns-cache-poisoning-0x20-cookies.md` |
| [SVCB and HTTPS Records: Service Binding in the DNS (RFC 9460 and 9461)](05h-svcb-https-records.md) | `05h-svcb-https-records.md` |
| [DNS Message Size: EDNS(0), the TC Bit and TCP Fallback](17-dns-message-size-edns-tcp-fallback.md) | `17-dns-message-size-edns-tcp-fallback.md` |
| [CNAME Chains and Answer Order: What the RFCs Say and What glibc Does](18-cname-chains-and-answer-order.md) | `18-cname-chains-and-answer-order.md` |
| [Anycast Services: One Address, Many Servers, and Withdrawing When Sick](19-anycast-services-and-dns.md) | `19-anycast-services-and-dns.md` |
| [NAT64, DNS64 and 464XLAT: Running IPv6-Only Without Losing the IPv4 Internet](21-nat64-dns64-464xlat.md) | `21-nat64-dns64-464xlat.md` |
| [IPv4 as a Service over IPv6: DS-Lite, MAP-E, MAP-T and 6rd](21b-ds-lite-map-e-map-t-6rd.md) | `21b-ds-lite-map-e-map-t-6rd.md` |
| [DHCPv6 Prefix Delegation: How a CPE Router Gets Its /56 and What It Must Do](22-dhcpv6-prefix-delegation.md) | `22-dhcpv6-prefix-delegation.md` |
| [Link-Local Service Discovery: mDNS, DNS-SD and Why the Printer Vanishes Across](24-mdns-dns-sd-service-discovery.md) | `24-mdns-dns-sd-service-discovery.md` |
