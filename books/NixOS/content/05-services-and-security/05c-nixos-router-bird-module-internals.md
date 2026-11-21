# NixOS Router: the typed BIRD module

**Goal:** Use the [stepbrobd/router](https://github.com/stepbrobd/router) flake to write BIRD 3 configuration as typed Nix, and understand how the options are generated so you can debug them.

> [!NOTE]
> This is the project behind the NixCon 2025 talk covered in [Internet-scale routing with BIRD and BGP](05b-internet-routing-bird-bgp.md). It is described from the project's README. The project is young: check the repository for current option names before you copy anything.

## What it is

NixOS Router compiles the **source grammar of a routing daemon** into a NixOS module. It generates configuration only. The daemon, its package, secrets and host networking stay in your own NixOS configuration.

| Fact | Detail |
|------|--------|
| Supported daemon | **BIRD** only, at the moment |
| Module | `nixosModules.bird` adds `services.bird.settings` |
| Version | In theory any BIRD 3 release; a build from another commit may lack a setting that the generated options contain |
| Recommended | Keep `services.bird.checkConfig` on (the upstream default): it runs `bird -p` on the generated file at build time |
| Licence | Compiler and runtime MIT; the generated inventory records provenance of BIRD's GPL-2.0-or-later grammar |
| Repository layout | `modules/` (`nixos`, `flake`), `lib/router/bird` (runtime), `registry/bird.toml`, `generated/`, `examples/bird`, `checks/bird` |

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

## Naming rules

- Each setting has **one path**. Multiword directives are dashed: `router-id`, `hold-time`, `peer-table`.
- Declarations are namespaced: `protocol.bgp.peer`.
- Records and blocks nest.
- **Keyword paths such as `hold.time` are rejected**, and the error tells you the canonical spelling (`hold-time`).
- Repeated entries are **ordered lists**.
- Executable bodies are native BIRD text.

## Functions and filters

Functions and filters are named records with native BIRD code in `body`.

```nix
{
  function.owned = {
    args = [{ name = "candidate"; type = "prefix"; }];
    return = "bool";
    variables = "int x;";            # optional locals before the body braces
    body = ''
      return candidate ~ [ 203.0.113.0/24 ];
    '';
  };

  filter.export_owned.body = ''
    if owned(net) then accept;
    reject;
  '';
}
```

| Rule | Detail |
|------|--------|
| `args` | Ordered list of `{ name, type }`; empty or omitted for `()` |
| `body` and `variables` | Merge as Nix lines. Put `mkBefore` and `mkAfter` on these fields, not on the declaration record |
| Other scalars | Ordinary module priorities and conflicts |
| Inline filter | `import.filter.body = "accept;";` |
| Named filter | `import.filter = "export_owned";` names an existing one |
| `where` clauses | Remain BIRD text |

## Typed literals

Arguments are typed data, not BIRD clauses. Text is quoted for you, and a plain string in an expression position is a string literal. Other literals need constructors:

| Constructor | Produces |
|-------------|----------|
| `bird.ip "192.0.2.1"` | An IP address |
| `bird.prefix "203.0.113.0/24"` | A prefix |
| `bird.tuple [ 65000 100 ]` | A tuple |
| `bird.ec [ "rt" 65000 100 ]` | An extended community (`rt`, with range and wildcard forms for EVPN and L3VPN) |
| `bird.symbol "NAME"` | A reference to a symbol |
| `bird.expr "A_VALUE + 1"` | Native expression code, passed through |

These also disambiguate slots that accept both text and addresses, such as a static route's `via`. Durations are `{ value = 100; unit = "ms"; }`.

## More shapes

```nix
{
  # keyword-led values are records
  ipv4.table.main4.cork-threshold = { low = 1000; high = 2000; };

  protocol.ospf.ospf0 = {
    ecmp = { enabled = true; limit = 8; };
    area."0.0.0.1" = {
      networks = [
        { prefix = "172.16.1.0/24"; }
        { prefix = "172.16.2.0/24"; hidden = true; }
      ];
      interface = [{
        pattern = "eth1";
        type = "nbma";
        neighbors = [
          { address = "192.168.120.1"; eligible = true; }
          { address = "192.168.120.2"; }
        ];
      }];
    };
  };

  # static routes: single next hop, multipath, MPLS label stack
  protocol.static.routes.route = [
    { prefix = "198.51.100.0/24"; via = bird.ip "192.0.2.2"; weight = 2; bfd = true; }
    { prefix = "10.0.0.0/8"; via = bird.ip "192.0.2.2"; mpls = [ 16 17 ]; }
    {
      prefix = "203.0.113.0/24";
      via = [
        { address = bird.ip "192.0.2.2"; weight = 1; }
        { address = bird.ip "192.0.2.3"; weight = 3; onlink = true; }
      ];
    }
  ];
}
```

Notes from the README:

- Values with no keyword before them (for example the two thresholds of `cork-threshold`) take their names from the `[[labels]]` table in `registry/bird.toml`.
- Interface statements and static routes are **ordered lists that keep duplicate prefixes**. An `exclude` applies within its own statement only, so an exclusion and its wildcard go in one `patterns` list.
- A pattern that is a constant or expression is `expression`, as in `interface = [{ expression = bird.expr "UPLINKS"; }]`.
- A flowspec prefix is a record under `flow4` or `flow6` keyed by component (`dst`, `proto`, `dport`, `tcp-flags`). BIRD sorts the components, so attribute order does not matter.
- `bird.ec` accepts BIRD's three forms: exact (`[ "rt" 65000 1 ]`), range (`[ "rt" 65000 2 10 ]`) and wildcard (`[ "rt" 65001 ]`).

> [!TIP]
> If the generator cannot disambiguate a setting, write the exact production node `{ _rule; _production; _args; }`. Prefer the normal paths first.

## How the options are generated

The compiler turns a BIRD source tree into `generated/bird/grammar.json` in four passes.

```text
 BIRD source tree
   1 assembly      configure.ac + Makefiles + .Y fragments, run BIRD's own m4
                   generators, add cf-lex.l keywords, SHA-256 fingerprint inputs
   2 Bison parse   %union, %token, %type, rules, mid-rule actions, $$ and $n
   3 registry      bind rules to ids and storage roles from registry/bird.toml,
                   name unlabeled values, drop rules no config needs
   4 surface       expand forms per scope: braces -> nested scope,
                   comma recursion -> list, keyword alternatives -> enum
 grammar.json  ->  lib/router/bird loads it with lib.importJSON
```

| Stage | Why it matters |
|-------|----------------|
| Grammar taken from BIRD itself | The options track BIRD's real syntax instead of a hand-written subset |
| Source fingerprint | Records exactly which BIRD files produced the inventory |
| Verification against defining files | Each registry entry is checked against its file, `%union` tag and referenced symbols |
| Fail-fast generation | Generation fails if a form lacks an attribute form, a field lacks a name, or a label matches no setting |

The **runtime** interprets the inventory in Nix: the option type merges by schema (statement lists concatenate, `body` and `variables` merge as `types.lines`, blocks and records recurse, scalars must agree after priorities). Rendering selects the forms whose path prefixes the attribute path, scores candidates by specificity, and requires **exactly one** rendered text. Root declarations follow the registry's `declaration_order`. BIRD's `bird -p` still performs the semantic check.

## Using it in a fleet

1. Pin the flake input and the BIRD package version together.
2. Put per-host values (router-id, local AS, neighbours) in host modules and shared policy (functions, filters) in one common module. See [Shared modules](../07-ops-and-fleet/05-shared-modules.md).
3. Add a [NixOS test](../07-ops-and-fleet/08-nixos-tests.md) with two VMs that peer over a virtual link and assert the expected routes. The NixCon 2026 talk [Peering Declaratively](../16-misc/11-nixcon-2026-operations-security-and-platforms.md) takes this approach before reaching a live upstream.
4. Keep BGP passwords out of the store with [sops-nix](02-sops-nix.md).

## Related material in the repository

The `docs/` folder holds the slide material (anycast, an Internet routing primer, and a Typst source), and `examples/bird` plus `checks/bird` hold working configurations and checks you can read as templates.

## Pitfalls

| Pitfall | Mitigation |
|---------|------------|
| Option missing for your BIRD build | Match the BIRD package to a release the options target |
| Keyword-style path rejected | Use the dashed canonical name from the error |
| Ambiguous string vs address | Use `bird.ip`, `bird.prefix` or `bird.expr` |
| Long exclusion lists misbehave | Keep exclusion and wildcard in one `patterns` list |
| Untested policy | Run `bird -p` via `checkConfig`, then a NixOS test |
