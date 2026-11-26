---
title: '11. NetOps CI Pipeline: Containerlab, Batfish, Nornir, and Suzieq'
---

# NetOps CI Pipeline: Containerlab, Batfish, Nornir, and Suzieq

Treat network changes like software releases: **generate** configs from a Source of Truth, **simulate** control-plane invariants statically with **Batfish**, **test** live routing convergence in an ephemeral **Containerlab** topology, **deploy** via **Nornir**, and **verify** post-change state with **Suzieq**.

## PR-Gated NetOps Pipeline

```text
Git PR (YAML intent / Jinja2 templates)
   ├── 1. Render configs & lint syntax
   ├── 2. Batfish static analysis (no loops, BGP sessions established, ACLs allow healthchecks)
   ├── 3. Containerlab + pytest/Nornir (boot virtual FRR/cEOS, verify OSPF/BGP + ping/failover)
   ├── 4. Merge -> Nornir/Ansible/gNMI atomic push to production
   └── 5. Suzieq snapshot diff (assert 0 unexpected down peers or missing routes)
```

## Example `pytest` Pre-Change Assertion (Batfish / Pybatfish)

```python
def test_no_undefined_references_and_bgp_up(bf):
    # Fail PR if any ACL/route-map references an undefined prefix-list
    undef = bf.q.undefinedReferences().answer().frame()
    assert undef.empty, f"Undefined config references found:\n{undef}"

    # Verify all configured BGP sessions establish in simulation
    bgp = bf.q.bgpSessionStatus().answer().frame()
    down = bgp[bgp["Established_Status"] != "ESTABLISHED"]
    assert down.empty, f"Unestablished BGP sessions:\n{down}"
```

## Acceptance Drills

- Open a PR that accidentally introduces a typo in a `route-map` prefix-list name; verify Batfish fails the PR in seconds before any router is touched.
- Run `suzieq-cli bgp assert --status=fail` after deployment to confirm zero broken adjacencies.
