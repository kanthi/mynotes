---
title: Internet-scale routing with BIRD and BGP
---

# Internet-scale routing with BIRD and BGP

**Goal:** Manage BGP sessions, routing policy and a small anycast service declaratively with the NixOS module system, instead of hand-edited `bird.conf` files.

> [!NOTE]
> This chapter is based on the NixCon 2025 talk [Internet scale routing with NixOS](https://talks.nixcon.org/nixcon-2025/talk/7YWTUC/) by Yifei Sun, an experience report. Code: [stepbrobd/router](https://github.com/stepbrobd/router). Slides: [slides.pdf](https://stepbrobd.github.io/router/slides.pdf).

For the module's typed literals, supported shapes and generator internals, continue with [NixOS Router: the typed BIRD module](05c-nixos-router-bird-module-internals.md).

## Why this chapter exists

[Firewall discipline](05-firewall-discipline.md) covers hosts that sit behind someone else's network. If you own an address block and an AS number, the host becomes a router. It speaks BGP to upstreams, announces prefixes, and decides which traffic leaves through which upstream. NixOS suits this well:

| Routing problem | NixOS answer |
|-----------------|--------------|
| Config drift between routers | One flake, one module set, many hosts |
| Typos in policy reach production | Build-time `bird -p` check |
| Per-host peers and prefixes | Module options and per-host overrides |
| Rollback after a bad announcement | Generations (see [deployment](../07-ops-and-fleet/00-index.md)) |

---

## Theory 1: the pieces

| Piece | Role |
|-------|------|
| **BGP** | Exchanges reachable prefixes between autonomous systems (AS) |
| **BIRD** (v3) | Routing daemon: protocols, filters, route tables |
| **systemd-networkd** | Addresses, tunnels, and policy-routing rules and tables on the host |
| **nftables** | Packet filtering, plus marking where policy routing needs it |
| **Anycast** | The same prefix announced from several hosts; routing picks the nearest |

The talk's scenario is a multi-upstream setup. Traffic from addresses inside your advertised prefixes has to leave through the upstream that announces them, or it can be dropped by the upstream's source-address checks. The fix is policy routing: choose the routing table by source address.

---

## Theory 2: BIRD settings as a NixOS module

The router project compiles BIRD's configuration grammar into NixOS options. It generates configuration only. The daemon package, secrets and host networking remain yours.

```nix
{ inputs, ... }:
{
  imports = [ inputs.router.nixosModules.bird ];

  services.bird = {
    enable = true;
    settings = {
      router-id = "192.0.2.1";
      protocol.device.device0 = { };
    };
  };
}
```

Conventions of the generated options:

| Rule | Example |
|------|---------|
| Multiword directives are dashed | `router-id`, `hold-time` |
| Declarations are namespaced | `protocol.bgp.peer` |
| Records and blocks nest | `neighbor = { address = "..."; as = 65001; }` |
| Executable bodies are native BIRD text | `filter.NAME.body = '' ... '';` |

> [!TIP]
> Keep `services.bird.checkConfig` enabled (the upstream default). It runs `bird -p` on the generated file at build time, so a broken policy fails `nixos-rebuild build` rather than the live session.

---

## Example 1: one BGP session with an export filter

Documentation address space is used throughout (`192.0.2.0/24`, `203.0.113.0/24`, private ASNs).

```nix
{
  services.bird.settings = {
    router-id = "192.0.2.1";

    function.owned = {
      args = [{ name = "candidate"; type = "prefix"; }];
      return = "bool";
      body = ''
        return candidate ~ [ 203.0.113.0/24 ];
      '';
    };

    filter.export_owned.body = ''
      if owned(net) then accept;
      reject;
    '';

    protocol.bgp.upstream_a = {
      local.as = 65000;
      neighbor = { address = "192.0.2.2"; as = 65001; };
      hold-time = 30;
      ipv4 = {
        import = "none";
        export.filter = "export_owned";
      };
    };
  };
}
```

Explanation: import nothing, export only your own prefix. This is the safe default for a leaf network. A small mistake in an export filter can leak routes to the whole internet.

---

## Example 2: sessions from a list of upstreams

Because settings are ordinary Nix values, generate sessions with `lib.mapAttrs`:

```nix
{ lib, ... }:
let
  upstreams = {
    upstream_a = { address = "192.0.2.2"; as = 65001; };
    upstream_b = { address = "198.51.100.2"; as = 65002; };
  };
in
{
  services.bird.settings.protocol.bgp = lib.mapAttrs (_: n: {
    local.as = 65000;
    neighbor = n;
    ipv4 = { import = "none"; export.filter = "export_owned"; };
  }) upstreams;
}
```

This is the main advantage over a flat config file: adding an upstream is one attribute, and policy cannot differ between sessions by accident.

---

## Example 3: source-based policy routing

Per the talk, host-side routing uses systemd-networkd and nftables rather than ad hoc `ip rule` commands. A sketch with networkd routing policy rules:

```nix
{
  systemd.network.networks."10-upstream-a" = {
    matchConfig.Name = "eth0";
    routingPolicyRules = [
      { From = "203.0.113.0/25"; Table = 100; }
    ];
    routes = [
      { Gateway = "192.0.2.2"; Table = 100; }
    ];
  };
}
```

Replies and forwarded traffic from that sub-prefix use table 100, and so leave via upstream A.

> [!WARNING]
> This sketch is illustrative. Match interface names, tables and prefixes to your own links, and test from a second vantage point before announcing.

---

## Example 4: a primitive anycast CDN

1. Announce the same `/24` from several hosts, each running BIRD with the module above.
2. Run the same service on every host (for example Caddy serving identical content, from the same flake).
3. Add a health gate: withdraw the prefix when the local service is down, for example with a BIRD protocol that depends on a check script, so that a dead node stops attracting traffic.

The talk calls this primitive on purpose. There is no load awareness, and TCP flows can shift mid-connection when routes change.

---

## Notes and pitfalls

| Pitfall | Mitigation |
|---------|------------|
| Leaking routes | Default `import = "none"`; explicit export filters; test with `bird -p` |
| Secrets (BGP passwords) | Use [sops-nix](02-sops-nix.md) paths, never Nix strings |
| Firewall blocks TCP/179 | Open it per interface and per peer, not globally (see [firewall discipline](05-firewall-discipline.md)) |
| Version mismatch | The generated options target BIRD 3; a package built from another release may lack a generated setting |
| Overlay networks (for example Tailscale) | The talk reports friction when combining them with policy routing; verify routes after each change |
| Reverse-path filtering | Multi-upstream setups may need loose `rp_filter` |

---

## Checklist

- [ ] Can explain why export filters default to deny
- [ ] BIRD config generated from the flake and checked at build time
- [ ] Every session's peer address and ASN lives in one reviewed place
- [ ] Source-based routing verified from outside the network
- [ ] BGP passwords come from runtime secret paths

## Additional resources

- [NixCon 2025 talk page](https://talks.nixcon.org/nixcon-2025/talk/7YWTUC/)
- [stepbrobd/router](https://github.com/stepbrobd/router)
- [BIRD documentation](https://bird.network.cz/?get_doc)
