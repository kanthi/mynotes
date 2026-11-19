---
title: 'Chapter 13: VPNs: WireGuard, IPsec, and Encrypted Tunnels'
---

# Chapter 13: VPNs: WireGuard, IPsec, and Encrypted Tunnels

Encrypted tunnels extend private networks safely across untrusted public infrastructure. On Linux, you have first-class, kernel-accelerated implementations of every major tunneling architecture: **WireGuard** (modern, stateless, Noise-protocol UDP tunnels), **IPsec / XFRM** with **strongSwan** (standards-based enterprise/cloud site-to-site VPNs compatible with Cisco, Palo Alto, AWS, and Azure), **OpenVPN** (TCP/UDP TLS tunnels with kernel DCO offload), and unencrypted **GRE / IPIP** overlays (often paired with IPsec for dynamic routing).

## Tunnel Encapsulation Overhead and MTU/MSS Math

Every tunnel wraps the original IP packet in extra headers. If you ignore encapsulation overhead, PMTUD breaks over bad firewalls and TLS handshakes stall!

| Tunnel Type | Outer Header Overhead (IPv4) | Recommended Tunnel MTU | IPv4 TCP MSS Clamp |
|---|---|---|---|
| **IPIP** | 20 B (IPv4) | `1480` | `1440` |
| **GRE** | 24 B (20 IPv4 + 4 GRE) | `1476` | `1436` |
| **WireGuard** | 60 B (20 IPv4 + 8 UDP + 32 WG) / 80 B (IPv6) | **`1420`** | `1380` |
| **IPsec ESP (AES-GCM)** | ~58–74 B (varies with NAT-T UDP 4500) | **`1400`** (safe conservative) | `1360` |

Clamp TCP MSS automatically on a Linux VPN gateway with `nftables` so inner TCP connections never exceed path capacity:

```nft
table inet mangle {
    chain forward {
        type filter hook forward priority mangle; policy accept;
        tcp flags syn / syn,rst tcp option maxseg size set rt mtu
    }
}
```

## WireGuard: Architecture and Cryptokey Routing

Merged directly into Linux kernel 5.6+, **WireGuard** replaces hundreds of thousands of lines of legacy crypto state machines with ~4,000 lines of auditable C using fixed, modern cryptography (Noise_IKpsk2, Curve25519, ChaCha20-Poly1305, BLAKE2s).

### Cryptokey Routing (`AllowedIPs`)

Every WireGuard interface (`wg0`) maintains a table mapping each peer's **Public Key** to its **`AllowedIPs`**. That single field performs **two roles simultaneously**:
1. **Outbound (Routing Selector):** When the Linux FIB routes a packet into `wg0`, WireGuard looks up the Destination IP in `AllowedIPs` to select which peer's public key encrypts it and which UDP `Endpoint` receives it.
2. **Inbound (Source ACL / uRPF):** When an encrypted packet arrives from a peer and decrypts cleanly, WireGuard checks that the inner **Source IP** matches that peer's `AllowedIPs` — otherwise it is dropped immediately!

> [!WARNING]
> **Preventing the `AllowedIPs = 0.0.0.0/0` routing loop:** If you route all traffic (`0.0.0.0/0`) into `wg0`, how do the encrypted outer UDP packets still reach the peer's public Endpoint over `eth0`? `wg-quick` solves this automatically using **fwmark + Policy-Based Routing**:
> - WireGuard stamps outgoing encrypted UDP packets with `FwMark = 0xca6c`.
> - `wg-quick` installs `ip rule add not fwmark 51820 table 51820` and places `default dev wg0` in table `51820` — so WireGuard's own encrypted outer packets bypass table `51820` and exit `eth0` cleanly via `main`!

### Site-to-Site WireGuard Configuration (`/etc/wireguard/wg0.conf`)

```ini
# Site A Gateway (/etc/wireguard/wg0.conf, chmod 600)
[Interface]
Address = 10.250.0.1/30
ListenPort = 51820
PrivateKey = <SITE_A_PRIVATE_KEY>
MTU = 1420

[Peer]
PublicKey = <SITE_B_PUBLIC_KEY>
PresharedKey = <OPTIONAL_POST_QUANTUM_PSK>
Endpoint = 203.0.113.2:51820
# Route Site B's tunnel IP AND Site B's internal LAN subnet
AllowedIPs = 10.250.0.2/32, 10.20.0.0/24
# Keep NAT mappings alive if behind a stateful firewall
PersistentKeepalive = 25
```

Manage and inspect WireGuard tunnels:

```bash
# Generate keypair safely (umask 077)
wg genkey | tee private.key | wg pubkey > public.key

# Start/enable via systemd
sudo systemctl enable --now wg-quick@wg0

# Inspect live peer endpoints, latest handshakes, and RX/TX bytes
sudo wg show
```

> [!TIP]
> **Dynamic routing (OSPF/BGP) over WireGuard:** For point-to-point site links carrying FRRouting OSPF/BGP, set `Table = off` in `[Interface]` (so `wg-quick` does not fight FRR for routes) and `AllowedIPs = 0.0.0.0/0, ::/0` on that single peer interface!

## Enterprise IPsec on Linux: strongSwan + Modern XFRM Interfaces

Traditional **Policy-Based IPsec** matches traffic using kernel XFRM policies (`ip xfrm policy`) with no dedicated network interface — making firewalling, `tcpdump`, policy routing, and BGP/OSPF painful.

Modern Linux kernels (4.19+) solve this with **XFRM Interfaces (`type xfrm`)** — the Linux equivalent of Cisco/Juniper **Route-Based VPNs (VTI)**!

```text
┌─────────────────────────────────────────────────────────────────────────┐
│               ROUTE-BASED IPSEC WITH LINUX XFRM INTERFACES              │
│                                                                         │
│   Application / Forwarded Packet (dst 10.20.0.5)                        │
│         │                                                               │
│         ▼                                                               │
│   Kernel FIB Route: 10.20.0.0/24 dev xfrm0                              │
│         │                                                               │
│         ▼                                                               │
│   [ xfrm0 (if_id = 42) ]  ◄── Standard net_device (nftables, BGP, MTU!) │
│         │                                                               │
│         ▼ (Matches XFRM Policy & SA with matching if_id = 42)           │
│   Kernel ESP Encryption (AES-256-GCM / hardware AES-NI)                 │
│         │                                                               │
│         ▼                                                               │
│   Outer IPv4/UDP-4500 packet exits physical eth0 to Peer Gateway        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Creating an XFRM Interface + strongSwan `swanctl.conf`

```bash
# Create route-based IPsec interface tied to if_id 42
sudo ip link add xfrm0 type xfrm dev eth0 if_id 42
sudo ip addr add 169.254.42.1/30 dev xfrm0
sudo ip link set xfrm0 mtu 1400 up
sudo ip route add 10.20.0.0/24 via 169.254.42.2 dev xfrm0
```

Corresponding modern **strongSwan** (`/etc/swanctl/conf.d/site-b.conf`):

```text
connections {
    site-b {
        version = 2
        local_addrs = 198.51.100.1
        remote_addrs = 203.0.113.2
        proposals = aes256gcm16-prfsha384-ecp384

        local {
            auth = psk
            id = site-a.example.com
        }
        remote {
            auth = psk
            id = site-b.example.com
        }

        children {
            site-b-tunnel {
                local_ts = 0.0.0.0/0
                remote_ts = 0.0.0.0/0
                esp_proposals = aes256gcm16-ecp384
                if_id_in = 42
                if_id_out = 42
                start_action = start
                dpd_action = restart
            }
        }
    }
}

secrets {
    ike-site-b {
        id-1 = site-a.example.com
        id-2 = site-b.example.com
        secret = "REPLACE_WITH_LONG_RANDOM_SECRET"
    }
}
```

Inspect live IPsec tunnels in both strongSwan and the Linux kernel:

```bash
sudo swanctl --load-all
sudo swanctl --list-sas
# Inspect kernel XFRM Security Associations (keys, SPIs, byte counters) & Policies
sudo ip -s xfrm state
sudo ip xfrm policy
```

## GRE, IPIP, and OpenVPN DCO

- **GRE (`type gre` / `gretap`):** Encapsulates IPv4, IPv6, multicast (OSPF `224.0.0.5`), and even Layer 2 Ethernet (`gretap`) over IP. Unencrypted on its own — wrap GRE inside IPsec transport mode when traversing public networks:
  ```bash
  sudo ip link add gre0 type gre local 198.51.100.1 remote 203.0.113.2 ttl 64
  sudo ip addr add 172.31.0.1/30 dev gre0
  sudo ip link set gre0 mtu 1476 up
  ```
- **OpenVPN with DCO (Data Channel Offload):** Historically OpenVPN copied every packet across user/kernel space via `tun0`, capping throughput. Modern Linux kernels (`ovpn-dco` / OpenVPN 2.6+) offload AES-GCM/ChaCha20 data-plane encryption directly into kernel space, matching WireGuard-class speeds while keeping TCP-443 firewall traversal and X.509 PKI compatibility.

## Hands-On Lab 13: Site-to-Site Encrypted WireGuard VPN Across a Simulated Internet

Connect two private LANs (`siteA-lan`: `10.10.0.0/24` and `siteB-lan`: `10.20.0.0/24`) through two Linux WireGuard gateways (`gwA` and `gwB`) separated by an untrusted `isp` router that has **zero routes** to `10.10.0.0/24` or `10.20.0.0/24`, and verify encrypted UDP/51820 packets on `isp`!

```bash
# 1. Create namespaces: siteA, gwA, isp, gwB, siteB
for ns in siteA gwA isp gwB siteB; do sudo ip netns add "$ns"; done

# 2. Wire: siteA <-> gwA <-> isp <-> gwB <-> siteB
sudo ip link add a-eth0  type veth peer name gwa-lan
sudo ip link add gwa-wan type veth peer name isp-a
sudo ip link add isp-b   type veth peer name gwb-wan
sudo ip link add gwb-lan type veth peer name b-eth0

sudo ip link set a-eth0  netns siteA; sudo ip link set gwa-lan netns gwA
sudo ip link set gwa-wan netns gwA;   sudo ip link set isp-a   netns isp
sudo ip link set isp-b   netns isp;   sudo ip link set gwb-wan netns gwB
sudo ip link set gwb-lan netns gwB;   sudo ip link set b-eth0  netns siteB

# 3. Configure underlay IPs and forwarding (ISP knows ONLY public WAN subnets!)
sudo ip -n siteA link set a-eth0 up; sudo ip -n siteA addr add 10.10.0.10/24 dev a-eth0
sudo ip -n siteA route add default via 10.10.0.1

sudo ip -n gwA link set gwa-lan up; sudo ip -n gwA addr add 10.10.0.1/24 dev gwa-lan
sudo ip -n gwA link set gwa-wan up; sudo ip -n gwA addr add 198.51.100.2/30 dev gwa-wan
sudo ip -n gwA route add default via 198.51.100.1
sudo ip netns exec gwA sysctl -w net.ipv4.ip_forward=1

sudo ip -n isp link set isp-a up; sudo ip -n isp addr add 198.51.100.1/30 dev isp-a
sudo ip -n isp link set isp-b up; sudo ip -n isp addr add 203.0.113.1/30 dev isp-b
sudo ip netns exec isp sysctl -w net.ipv4.ip_forward=1

sudo ip -n gwB link set gwb-wan up; sudo ip -n gwB addr add 203.0.113.2/30 dev gwb-wan
sudo ip -n gwB link set gwb-lan up; sudo ip -n gwB addr add 10.20.0.1/24 dev gwb-lan
sudo ip -n gwB route add default via 203.0.113.1
sudo ip netns exec gwB sysctl -w net.ipv4.ip_forward=1

sudo ip -n siteB link set b-eth0 up; sudo ip -n siteB addr add 10.20.0.20/24 dev b-eth0
sudo ip -n siteB route add default via 10.20.0.1

# 4. Generate ephemeral WireGuard keypairs for gwA and gwB
mkdir -p /tmp/lab13-wg
chmod 700 /tmp/lab13-wg
wg genkey | tee /tmp/lab13-wg/a.priv | wg pubkey > /tmp/lab13-wg/a.pub
wg genkey | tee /tmp/lab13-wg/b.priv | wg pubkey > /tmp/lab13-wg/b.pub
PUB_A=$(cat /tmp/lab13-wg/a.pub)
PUB_B=$(cat /tmp/lab13-wg/b.pub)

# 5. Create native kernel WireGuard interfaces inside gwA and gwB
sudo ip -n gwA link add wg0 type wireguard
sudo ip -n gwA addr add 10.250.0.1/30 dev wg0
sudo ip netns exec gwA wg set wg0 listen-port 51820 private-key /tmp/lab13-wg/a.priv \
  peer "$PUB_B" endpoint 203.0.113.2:51820 allowed-ips 10.250.0.2/32,10.20.0.0/24
sudo ip -n gwA link set wg0 mtu 1420 up
sudo ip -n gwA route add 10.20.0.0/24 dev wg0

sudo ip -n gwB link add wg0 type wireguard
sudo ip -n gwB addr add 10.250.0.2/30 dev wg0
sudo ip netns exec gwB wg set wg0 listen-port 51820 private-key /tmp/lab13-wg/b.priv \
  peer "$PUB_A" endpoint 198.51.100.2:51820 allowed-ips 10.250.0.1/32,10.10.0.0/24
sudo ip -n gwB link set wg0 mtu 1420 up
sudo ip -n gwB route add 10.10.0.0/24 dev wg0

# 6. Capture traffic on the untrusted ISP router while siteA pings siteB!
#    ISP sees ONLY encrypted UDP/51820 packets between 198.51.100.2 and 203.0.113.2 — zero plaintext ICMP!
sudo ip netns exec isp timeout 3 tcpdump -l -nn -i isp-a &
sleep 0.5
sudo ip netns exec siteA ping -c 3 10.20.0.20
wait

# 7. Inspect WireGuard handshake and byte counters on gwA
sudo ip netns exec gwA wg show

# 8. Cleanup
rm -rf /tmp/lab13-wg
for ns in siteA gwA isp gwB siteB; do sudo ip netns del "$ns"; done
```

**What you verified:** Private subnets `10.10.0.10` and `10.20.0.20` communicated seamlessly across an untrusted ISP that had no knowledge of their routes and could see only encrypted UDP/51820 WireGuard datagrams.
