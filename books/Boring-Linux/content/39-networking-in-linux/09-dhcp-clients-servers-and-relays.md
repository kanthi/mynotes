---
title: 'Chapter 9: DHCP: Clients, Servers, and Relays'
---

# Chapter 9: DHCP: Clients, Servers, and Relays

Dynamic Host Configuration Protocol (DHCPv4 over UDP 67/68 and DHCPv6 over UDP 546/547) automates IP addressing, default gateways, static routes (RFC 3442 Option 121), DNS resolvers, NTP servers, MTU, and PXE network booting. On Linux, you operate DHCP in three roles: **client**, **authoritative server** (ISC Kea or `dnsmasq`), and **DHCP relay agent** bridging remote subnets across routers.

## DHCPv4 DORA and DHCPv6 SARR Wire Mechanics

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                    DHCPv4 DORA & LEASE LIFECYCLE                        │
│                                                                         │
│   Client (0.0.0.0:68)                         Server (x.x.x.x:67)       │
│       │                                                │                │
│       ├─── 1. DHCPDISCOVER (Broadcast 255.255.255.255)►│                │
│       │◄── 2. DHCPOFFER    (Offered IP + options) ─────┤                │
│       ├─── 3. DHCPREQUEST  (Broadcast: accept server) ►│                │
│       │◄── 4. DHCPACK      (Lease committed + T1/T2) ──┤                │
│       │                                                │                │
│       │   [50% lease (T1): Unicast DHCPREQUEST renew]  │                │
│       │   [87.5% lease (T2): Broadcast rebind]         │                │
└─────────────────────────────────────────────────────────────────────────┘
```

### DHCPv6: Stateless vs Stateful vs Prefix Delegation (PD)

In IPv6, Router Advertisements (RAs) carry two critical bits that tell Linux hosts how to interact with DHCPv6:
- **`M` (Managed) = 0, `O` (Other) = 0:** Pure SLAAC (host self-assigns IPv6 and gets DNS via RDNSS in the RA).
- **`M` = 0, `O` = 1:** **Stateless DHCPv6** (SLAAC provides the address; DHCPv6 provides DNS/search domains/NTP).
- **`M` = 1:** **Stateful DHCPv6** (SARR: `Solicit → Advertise → Request → Reply` using `IA_NA` for host addresses, or `IA_PD` for delegating whole subnets like a `/56` or `/60` to downstream Linux routers).

## Linux DHCP Clients

| Client | Where Used | Lease / Inspect Command |
|---|---|---|
| **`systemd-networkd`** (built-in) | Ubuntu Server (via Netplan), servers, containers | `networkctl status <iface>` and `/run/systemd/netif/leases/<ifindex>` |
| **`NetworkManager`** (built-in) | RHEL/Fedora, workstations | `nmcli dev show <iface>` |
| **`dhcpcd`** | Embedded, Debian/Alpine/Arch routers, IPv6 PD | `dhcpcd -U <iface>` |
| **`dhclient`** (ISC, legacy EOL) | Older scripts & quick namespace lab tests | `/var/lib/dhcp/dhclient.leases` |

### Controlling Client Identifiers (DUID vs MAC)

> [!WARNING]
> **DHCP reservation surprise on Ubuntu / `systemd-networkd`:** By default, `systemd-networkd` sends an RFC 4361 **IAID + DUID** as DHCPv4 Option 61 (Client Identifier) instead of the raw Ethernet MAC address! If your corporate DHCP server matches static reservations by MAC address, cloned VMs may pull different IPs unless you set `ClientIdentifier=mac` (in `systemd-networkd`) or `dhcp-identifier: mac` (in Netplan):
> ```yaml
> ethernets:
>   enp1s0:
>     dhcp4: true
>     dhcp-identifier: mac
> ```

### Ignoring Unwanted Server Options (e.g., Default Route or DNS)

On a multi-NIC server where only WAN should set the default gateway and DNS:

```ini
# /etc/systemd/network/20-lan.network
[Match]
Name=enp2s0

[Network]
DHCP=ipv4

[DHCPv4]
ClientIdentifier=mac
UseDNS=no
UseRoutes=no
UseHostname=no
```

## Production DHCP Servers on Linux: ISC Kea vs `dnsmasq`

Legacy ISC `dhcpd` reached End of Life in 2022. Modern Linux servers use:
- **ISC Kea (`kea-dhcp4`, `kea-dhcp6`)**: Enterprise-grade, multi-threaded, JSON-configured, REST API control socket, PostgreSQL/MySQL lease backends, High-Availability active-active/hot-standby failover, and Dynamic DNS (`kea-dhcp-ddns`).
- **`dnsmasq`**: Lightweight, single-binary combined DHCP + TFTP + caching DNS server — unbeatable for branch routers, homelabs, hypervisor bridges (`libvirt`), and PXE provisioning.

### Production ISC Kea Configuration (`/etc/kea/kea-dhcp4.conf`)

```json
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": [ "enp1s0.10" ],
      "dhcp-socket-type": "raw"
    },
    "lease-database": {
      "type": "memfile",
      "persist": true,
      "name": "/var/lib/kea/kea-leases4.csv",
      "lfc-interval": 3600
    },
    "valid-lifetime": 28800,
    "renew-timer": 14400,
    "rebind-timer": 25200,
    "option-data": [
      { "name": "domain-name-servers", "data": "10.10.10.53, 10.10.10.54" },
      { "name": "domain-name", "data": "corp.internal" }
    ],
    "subnet4": [
      {
        "id": 10,
        "subnet": "10.10.10.0/24",
        "pools": [ { "pool": "10.10.10.100 - 10.10.10.220" } ],
        "option-data": [
          { "name": "routers", "data": "10.10.10.1" }
        ],
        "reservations": [
          {
            "hw-address": "52:54:00:11:22:33",
            "ip-address": "10.10.10.20",
            "hostname": "db-primary"
          }
        ]
      },
      {
        "id": 20,
        "subnet": "10.10.20.0/24",
        "pools": [ { "pool": "10.10.20.100 - 10.10.20.220" } ],
        "option-data": [
          { "name": "routers", "data": "10.10.20.1" }
        ]
      }
    ]
  }
}
```

Validate syntax before restarting Kea:

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
sudo systemctl enable --now kea-dhcp4-server
# Inspect live leases directly from Kea's CSV lease database
cat /var/lib/kea/kea-leases4.csv
```

### Lightweight `dnsmasq` DHCP + Static Leases + PXE Example

```ini
# /etc/dnsmasq.d/lab-dhcp.conf
interface=br0
bind-interfaces
# Disable DNS if running Unbound/BIND separately
port=0

# Subnet pool: start, end, netmask, lease time
dhcp-range=10.50.0.100,10.50.0.200,255.255.255.0,12h

# Options: 3=Default Gateway, 6=DNS, 15=Domain, 121=Classless Static Route
dhcp-option=option:router,10.50.0.1
dhcp-option=option:dns-server,10.50.0.1
dhcp-option=option:domain-name,lab.internal

# Static reservation by MAC
dhcp-host=52:54:00:aa:bb:cc,10.50.0.50,app01,infinite

# PXE / UEFI network boot support
dhcp-match=set:efi-x86_64,option:client-arch,7
dhcp-boot=tag:efi-x86_64,grubx64.efi,,10.50.0.5
```

## DHCP Relay Agents (`ip helper-address` on Linux)

Because `DHCPDISCOVER` is a Layer 2/3 broadcast (`255.255.255.255`), routers do not forward it across subnets. Instead, a **DHCP Relay Agent** on the router listens for broadcasts on client-facing VLAN interfaces, stamps its own ingress interface IP into the **`giaddr`** (Gateway IP Address) field of the DHCP payload, optionally appends **Option 82** (Relay Agent Information: Circuit ID / Remote ID), and unicasts the packet to the central DHCP server!

```text
  [Client VLAN 20] ──Broadcast DHCPDISCOVER──► [Linux Router: dhcrelay / dhcp-helper]
                                                stamps giaddr = 10.10.20.1
                                                       │
                                                Unicast UDP/67
                                                       ▼
                                       [Central Kea/dnsmasq Server: 10.10.10.5]
                                       (Matches giaddr 10.10.20.1 → Subnet 20 pool!)
```

Linux DHCP relay tools:
- **`dhcp-helper`** (lightweight, from the author of `dnsmasq`): `dhcp-helper -s 10.10.10.5 -i eth1`
- **`isc-dhcp-relay` (`dhcrelay`)**: `dhcrelay -4 -i eth1 -i eth0 10.10.10.5`
- **Firewall note:** Remember to permit **UDP port 67** inbound on the client interface (broadcast `255.255.255.255`) AND unicast UDP 67 between router and DHCP server!

## Hands-On Lab 9: Live DHCP Server + Static Reservation + Packet Capture

Run a real `dnsmasq` DHCP server inside namespace `dhcp-srv`, boot a client in namespace `client` that obtains a dynamic lease and gateway over a `veth` link, inspect the DORA handshake on the wire, and verify lease files.

```bash
# 1. Create namespaces and wire them
sudo ip netns add dhcp-srv
sudo ip netns add client
sudo ip link add srv0 type veth peer name cli0
sudo ip link set srv0 netns dhcp-srv
sudo ip link set cli0 netns client

# 2. Configure the DHCP server interface
sudo ip -n dhcp-srv link set lo up
sudo ip -n dhcp-srv link set srv0 up
sudo ip -n dhcp-srv addr add 10.90.0.1/24 dev srv0

# Give client a deterministic MAC for static reservation testing
sudo ip -n client link set lo up
sudo ip -n client link set cli0 address 02:90:00:00:00:42 up

# 3. Launch an isolated dnsmasq DHCP-only instance inside dhcp-srv
mkdir -p /tmp/lab9-dhcp
sudo ip netns exec dhcp-srv dnsmasq \
  --interface=srv0 \
  --bind-interfaces \
  --port=0 \
  --dhcp-range=10.90.0.100,10.90.0.150,255.255.255.0,1h \
  --dhcp-option=option:router,10.90.0.1 \
  --dhcp-option=option:dns-server,10.90.0.1 \
  --dhcp-host=02:90:00:00:00:42,10.90.0.42,lab-client \
  --dhcp-leasefile=/tmp/lab9-dhcp/leases \
  --pid-file=/tmp/lab9-dhcp/dnsmasq.pid \
  --log-dhcp

# 4. Capture the 4-way DORA handshake while requesting a lease from client
sudo ip netns exec dhcp-srv timeout 5 tcpdump -l -nn -i srv0 'port 67 or port 68' &
sleep 0.5
# Use udhcpc / dhclient / dhcpcd depending on installed client
if command -v dhclient >/dev/null 2>&1; then
  sudo ip netns exec client dhclient -1 -v -lf /tmp/lab9-dhcp/cli.lease -pf /tmp/lab9-dhcp/cli.pid cli0
elif command -v dhcpcd >/dev/null 2>&1; then
  sudo ip netns exec client dhcpcd -1 cli0
fi
wait

# 5. Verify client received static reservation 10.90.0.42/24 + default route via 10.90.0.1!
sudo ip -n client -br -c addr show dev cli0
sudo ip -n client route show
cat /tmp/lab9-dhcp/leases

# 6. Cleanup (stop dnsmasq, remove temp files, delete namespaces)
sudo kill "$(cat /tmp/lab9-dhcp/dnsmasq.pid)" 2>/dev/null || true
sudo kill "$(cat /tmp/lab9-dhcp/cli.pid)" 2>/dev/null || true
rm -rf /tmp/lab9-dhcp
sudo ip netns del dhcp-srv
sudo ip netns del client
```

**What you verified:** You captured the full UDP 68→67 DORA broadcast/unicast transaction, observed Option 3 (router) installing the default route in the client namespace, and matched a MAC-based static reservation in the lease file.
