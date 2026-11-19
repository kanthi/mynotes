---
title: Containerlab, netlab, and Linux Cheat Sheet
---

# Containerlab, netlab, and Linux Cheat Sheet

This reference covers the three toolsets you use every day in network labs: **Containerlab** for topology lifecycle and link impairment, **netlab** for multi-vendor lab automation and validation, and **Linux network utilities** for host/router data-plane inspection and packet capture.

## Containerlab

Run Containerlab commands from the directory containing your `*.clab.yml` file (or pass `-t <file>`).

```sh
# Deploy (or redeploy cleanly, wiping previous lab state)
sudo containerlab deploy -t topology.clab.yml
sudo containerlab deploy -t topology.clab.yml --reconfigure

# Inspect running nodes, management IPs, and container names
sudo containerlab inspect -t topology.clab.yml
sudo containerlab inspect --all

# Visualize topology in a browser or export DOT/Mermaid
sudo containerlab graph -t topology.clab.yml

# Save running configs from supported NOS containers to lab directory
sudo containerlab save -t topology.clab.yml

# Tear down the lab (--cleanup also removes the clab-<name> directory)
sudo containerlab destroy -t topology.clab.yml
sudo containerlab destroy -t topology.clab.yml --cleanup
```

### Link impairment (`tools netem`) and dynamic links (`tools veth`)

```sh
# Inject 50ms delay, 10ms jitter, and 2% packet loss on a container interface
sudo containerlab tools netem set -n clab-LAB-r1 -i eth1 --delay 50ms --jitter 10ms --loss 2

# Inspect active impairments on a node
sudo containerlab tools netem show -n clab-LAB-r1

# Clear all impairments on an interface
sudo containerlab tools netem reset -n clab-LAB-r1 -i eth1

# Hot-plug a new veth link between two running containers
sudo containerlab tools veth create -a clab-LAB-r1:eth3 -b clab-LAB-r2:eth3
```

### Accessing containers

```sh
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
docker exec -it clab-LAB-h1 bash
docker exec -it clab-LAB-r1 vtysh
docker exec -it clab-LAB-sw1 Cli
docker exec -it clab-LAB-srl1 sr_cli
```

> [!WARNING]
> `containerlab destroy --cleanup` deletes the generated `clab-<name>/` directory, including saved startup configs. Export or commit any config changes you want to keep before running `--cleanup`.

## netlab

`netlab` compiles a high-level `topology.yml` into provider configs (Containerlab, libvirt, etc.), generates Ansible inventories, pushes base routing/VLAN/BGP configs, and runs automated validation tests.

```sh
# Start the lab (create + deploy + initial device configuration)
netlab up

# Check running nodes, management IPs, and provider state
netlab status

# Connect interactively to a node (SSH or docker exec automatically)
netlab connect r1

# Run validation tests defined in topology.yml
netlab validate

# Capture packets directly from a node interface (streams to stdout/Wireshark)
netlab capture r1 eth1

# Tear down the lab and clean generated state
netlab down
netlab down --cleanup
```

## Linux Interfaces, Routing, VRFs, and Bridges (`ip`, `bridge`)

### `ip` (links, addresses, routes, neighbors, VRFs, namespaces)

```sh
# Concise interface and address summaries
ip -br link
ip -br -4 addr
ip -br -6 addr

# Detailed link info (MTU, VLAN, VXLAN, VRF master) and error counters
ip -d link show eth1
ip -s -s link show eth1

# IPv4/IPv6 routing tables and exact forwarding lookup
ip -4 route show
ip -6 route show
ip route get 192.0.2.1
ip -6 route get 2001:db8::1

# Policy routing rules and custom tables
ip rule show
ip route show table 100

# VRFs
ip vrf show
ip route show vrf RED
ip vrf exec RED ping -c 3 10.20.1.2

# ARP and IPv6 Neighbor Discovery tables
ip -4 neigh show
ip -6 neigh show
ip neigh flush dev eth1

# Network namespaces
ip netns list
ip netns exec ns1 ip -br addr
```

### `bridge` (VLAN-aware Linux bridges and FDB)

```sh
bridge link show
bridge vlan show
bridge fdb show
bridge fdb show dev vxlan10
```

## Sockets, NICs, QoS, and Firewalling (`ss`, `ethtool`, `tc`, `nft`)

### `ss` (listening sockets and active flows)

```sh
ss -tulpn                     # All listening TCP/UDP sockets with PIDs
ss -tan state established     # Active TCP sessions
ss -ti dst 10.1.12.2          # Detailed TCP metrics (cwnd, rtt, retrans)
```

### `ethtool` (link speed, offloads, driver stats)

```sh
ethtool eth1                  # Speed, duplex, link detected
ethtool -k eth1               # Offload features (TSO, GSO, GRO, checksum)
ethtool -S eth1               # Driver-level RX/TX drop/error counters
```

### `tc` (traffic control, shaping, and `netem`)

```sh
# Inspect qdiscs and drop/overlimit counters
tc -s qdisc show dev eth1

# Add 40ms delay and 1% loss directly on Linux
sudo tc qdisc add dev eth1 root netem delay 40ms loss 1%

# Apply a 10 Mbps token-bucket rate limit
sudo tc qdisc replace dev eth1 root tbf rate 10mbit burst 32kbit latency 50ms

# Remove custom qdisc
sudo tc qdisc del dev eth1 root
```

### `nft` (nftables firewall & NAT)

```sh
sudo nft list ruleset         # Full active ruleset with counters
sudo nft list tables          # List tables only
sudo nft -a list chain inet filter input   # Show rule handles for deletion
sudo nft reset counters       # Zero all counters before a test
```

## Packet Capture (`tcpdump`, `tshark`)

```sh
# Capture on eth1 without DNS resolution, full packets, verbose
sudo tcpdump -ni eth1 -s 0 -vv

# Filter common control-plane protocols
sudo tcpdump -ni eth1 'tcp port 179'                 # BGP
sudo tcpdump -ni eth1 'proto ospf'                   # OSPF
sudo tcpdump -ni eth1 'arp or icmp or icmp6'         # ARP, ICMPv4, ICMPv6/ND
sudo tcpdump -ni eth1 'udp port 4789'                # VXLAN
sudo tcpdump -ni eth1 'vlan'                         # 802.1Q tagged frames

# Write to a pcap file for Wireshark or tshark analysis
sudo tcpdump -ni eth1 -w /tmp/capture.pcap

# Quick CLI dissection with tshark
tshark -r /tmp/capture.pcap -Y "bgp or ospf"
tshark -r /tmp/capture.pcap -T fields -e frame.time_relative -e ip.src -e ip.dst -e _ws.col.Info
```

## Path, Throughput, DNS, and HTTP Checks (`mtr`, `iperf3`, `dig`, `curl`)

```sh
# Continuous hop-by-hop latency and packet loss report (10 cycles, numeric)
mtr -n -r -c 10 192.0.2.1

# Throughput testing (server on one host, client on another)
iperf3 -s                                     # Server
iperf3 -c 10.10.20.10 -P 4 -t 10              # Client: 4 parallel TCP streams, 10s
iperf3 -c 10.10.20.10 -u -b 100M -t 10        # Client: UDP 100 Mbps (measures jitter/loss)

# DNS resolution and trace
dig @10.53.0.10 example.test A +noall +answer
dig example.test +trace

# HTTP/TLS timing and headers
curl -svI https://service.example.test/
curl -o /dev/null -s -w "dns=%{time_namelookup}s tcp=%{time_connect}s tls=%{time_appconnect}s ttfb=%{time_starttransfer}s total=%{time_total}s\n" https://service.example.test/
```
