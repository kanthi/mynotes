---
title: Security Backporting, Vulnerability Triage, and Supply Chain Provenance
---

# Security Backporting, Vulnerability Triage, and Supply Chain Provenance

**Goal:** Establish rigorous vulnerability triage and security backporting practices in Nix, generating authoritative Software Bills of Materials (SBOMs in SPDX and CycloneDX), formulating Vulnerability Exploitability eXchange (VEX) statements, and securing binary artifact distribution with cryptographic signing and SLSA-compliant provenance.

## Why it matters

Modern software supply-chain security requires knowing exactly what code is running, which vulnerabilities affect it, and how fixes are deployed. In traditional distributions, vulnerability scanners (like Grype, Trivy, or Clair) guess installed components by inspecting package manager databases or matching file paths—often producing high rates of false positives and false negatives.

In Nix, every package is a cryptographic derivation with a known, closed dependency graph. When a zero-day CVE strikes (such as in OpenSSL, Curl, or XZ), packaging engineers cannot always bump to the latest upstream release—major version jumps introduce breaking API changes that would disrupt production systems. Instead, engineers must **backport security patches** cleanly to existing, pinned packages, generate verifiable SBOMs, author VEX advisories, and sign binary closures so customers and downstream users can verify integrity cryptographically.

---

## The Security Backporting Workflow

Security backporting is the art of extracting a patch that fixes a vulnerability from an upstream development branch and applying it to an older, stable package release while preserving ABI compatibility and avoiding mass rebuild cascades.

```text
  Upstream CVE Fix (Committed to upstream main/v3.4.x)
       │
       ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ 1. Patch Extraction & Cleanroom Isolation                     │
  │ Extract git diff, remove unrelated refactors/formatting      │
  └──────────────────────────────┬───────────────────────────────┘
                                 │
                                 ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ 2. Context Adjustment & Rebase                               │
  │ Adapt hunk line numbers and header changes to stable target  │
  └──────────────────────────────┬───────────────────────────────┘
                                 │
                                 ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ 3. Blast Radius Minimization                                 │
  │ Surgical overrideAttrs vs package set overlay evaluation     │
  └──────────────────────────────┬───────────────────────────────┘
                                 │
                                 ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ 4. Supply Chain Artifact Generation                          │
  │ Output SBOM (SPDX/CycloneDX), VEX statement, and Sigstore attestation │
  └──────────────────────────────────────────────────────────────┘
```

### Minimizing Rebuild Blast Radius

When a critical security flaw affects a foundational library like `zlib` or `openssl`, applying an overlay at the root of `pkgs` causes every dependent package in the entire OS to rebuild from source:

```text
Global Overlay:
pkgs.openssl (patched) ──> Rebuilds curl ──> Rebuilds git ──> Rebuilds python ──> Rebuilds 15,000 packages!
```

To deliver a security fix immediately without waiting for a 12-hour full cluster compilation:

1. **Surgical Call-Site Replacement:** If only specific outward-facing services (like an ingress proxy or edge gateway) expose the vulnerable vector, patch those specific instances first.
2. **Upstream Staging Coordination:** In Nixpkgs, changes with high rebuild counts are committed to `staging` rather than `master`, allowing Hydra to pre-build binary caches before channel release.
3. **Downstream Cache Injection:** Downstream builders build the patched closure on high-throughput build farms, sign the resulting NARs, and push them to binary caches so consumer hosts substitute pre-compiled binaries instantly.

### Worked Example: Surgical CVE Backport

```nix
# packages/security/cve-2026-openssl-backport.nix
{ lib, fetchpatch, openssl_3 }:

openssl_3.overrideAttrs (oldAttrs: rec {
  pname = "${oldAttrs.pname}-cve-patched";
  version = oldAttrs.version;

  patches = (oldAttrs.patches or []) ++ [
    # Apply official CVE fix backported to 3.0.x branch
    (fetchpatch {
      name = "CVE-2026-38491-asn1-buffer-overflow.patch";
      url = "https://git.openssl.org/gitweb/?p=openssl.git;a=patch;h=9a1b2c3d4e5f6a7b8c9d0e1f";
      hash = "sha256-R4h8J2x1vF8N3m7kL0pQ9sT6wU2yZ5aC8eB1dG4jH7k=";
      # Strip commit message noise to prevent patch fuzz failures
      stripLen = 1;
    })
  ];

  # Validate that the patch actually compiled the fix by running internal test suites
  checkFlags = (oldAttrs.checkFlags or []) ++ [
    "TESTS=test_asn1_encode test_ssl_cert"
  ];

  passthru = (oldAttrs.passthru or {}) // {
    cveMitigations = [ "CVE-2026-38491" ];
    advisoryUrl = "https://nvd.nist.gov/vuln/detail/CVE-2026-38491";
  };
})
```

---

## Vulnerability Feeds and Data Standards: CVE, OSV, and VEX

Security teams track vulnerabilities across multiple standard formats. A modern packaging platform ingests these feeds and correlates them against Nix derivations:

| Standard | Full Name | Primary Purpose in Packaging |
|----------|-----------|------------------------------|
| **CVE** | Common Vulnerabilities and Exposures | Universal catalog of disclosed cybersecurity vulnerabilities (NIST NVD) |
| **OSV** | Open Source Vulnerabilities | Machine-readable JSON schema optimized for package managers and Git commit ranges |
| **VEX** | Vulnerability Exploitability eXchange | Machine-readable assertions declaring whether a CVE is actually exploitable in a specific artifact |
| **SPDX** | System Package Data Exchange | ISO standard for Software Bill of Materials (SBOM) detailing packages, files, and licenses |
| **CycloneDX** | OWASP CycloneDX | Lightweight, security-focused SBOM standard with native support for VEX, vulnerabilities, and services |

### Ingesting OSV Vulnerability Records

The OSV schema provides precise Git commit and version ranges, making it ideal for automated Nix tooling:

```json
{
  "schema_version": "1.6.0",
  "id": "OSV-2026-4412",
  "summary": "Heap buffer overflow in libpng progressive reader",
  "package": {
    "name": "libpng",
    "ecosystem": "OSS-Fuzz"
  },
  "affected": [
    {
      "ranges": [
        {
          "type": "SEMVER",
          "events": [
            { "introduced": "1.6.0" },
            { "fixed": "1.6.44" }
          ]
        }
      ]
    }
  ]
}
```

A downstream packaging tool evaluates the installed Nix package version against the `affected` ranges in OSV.

---

## Ground-Truth SBOM Generation in Nix

Traditional SBOM generators scan filesystem folders and make heuristic guesses about library versions. In Nix, the derivation graph (`.drv`) and the runtime closure graph provide **100% deterministic ground truth**.

```text
  /nix/store/a7x8...-my-app.drv (Evaluation / Build-Time Graph)
       │
       ▼
  /nix/store/b9y2...-my-app (Realized Runtime Closure)
       │
       ├─► /nix/store/c1z3...-glibc-2.40
       ├─► /nix/store/d4w5...-openssl-3.0.15
       └─► /nix/store/e6v7...-zlib-1.3.1
```

### Generating SPDX and CycloneDX from Nix Closures

Using tools like `nix-sbom` or built-in derivation query commands, you can generate comprehensive SBOMs:

```bash
# Query all runtime dependencies of a package
nix path-info --recursive --json .#packages.x86_64-linux.my-service > closure.json

# Extract exact package names, versions, and store hashes into SPDX format
jq '[.[] | {
  SPDXID: ("SPDXRef-Package-" + .registrationTime),
  name: (.path | split("-")[1:] | join("-")),
  versionInfo: (.path | capture("-(?<v>[0-9].*)") | .v),
  downloadLocation: "NOASSERTION",
  filesAnalyzed: false,
  checksums: [{
    algorithm: "SHA256",
    checksumValue: .narHash
  }]
}]' closure.json > sbom.spdx.json
```

---

## Formulating VEX Statements to Eliminate False Positives

Vulnerability scanners frequently flag false positives. For example, a package may contain vulnerable C source code that is:
1. Disabled via configure flags (e.g. `--disable-vulnerable-protocol`)
2. Dead-code eliminated by the compiler during link-time optimization (LTO)
3. Patched downstream via `overrideAttrs` without altering the upstream version number string

A **VEX document** (OpenVEX or CycloneDX VEX) allows the packaging engineer to state authoritatively that a CVE is not exploitable:

```json
{
  "@context": "https://openvex.dev/ns/v0.2.0",
  "@id": "https://packages.corp.net/vex/2026-0012",
  "author": "Security Packaging Team <security@corp.net>",
  "timestamp": "2026-11-20T10:00:00Z",
  "version": 1,
  "statements": [
    {
      "vulnerability": {
        "name": "CVE-2026-2189"
      },
      "products": [
        {
          "@id": "pkg:nix/openssl@3.0.15?hash=sha256-R4h8J2x1vF8N3m7kL0pQ9sT6wU2yZ5aC8eB1dG4jH7k="
        }
      ],
      "status": "not_affected",
      "justification": "vulnerable_code_cannot_be_controlled_by_adversary",
      "impact_statement": "The vulnerable QUIC handshake parser is disabled at build time via configureFlags = [ \"no-quic\" ]; the binary does not contain the affected symbols."
    },
    {
      "vulnerability": {
        "name": "CVE-2026-38491"
      },
      "products": [
        {
          "@id": "pkg:nix/openssl-cve-patched@3.0.15"
        }
      ],
      "status": "fixed",
      "action_statement": "Mitigated by downstream patch CVE-2026-38491-asn1-buffer-overflow.patch applied during patchPhase."
    }
  ]
}
```

---

## Binary Cache Cryptographic Signing & SLSA Provenance

Distributing binaries requires cryptographic trust. If an attacker intercepts your binary cache or compromises a CI runner, they could inject malicious binaries into downstream fleets.

### Binary Cache Key Generation and Signing

Nix natively supports asymmetric public-key cryptography for binary caches:

```bash
# Generate binary cache private and public signing keys
nix-store --generate-binary-cache-key downstream-cache-1 secret-key public-key

# Inspect public key format
cat public-key
# Output: downstream-cache-1:wK7b3...=

# Sign a built store path before publishing to the cache
nix sign-paths --key-file secret-key ./result
```

When client machines fetch from the downstream substituter, they verify the signature against the trusted public key declared in their Nix configuration:

```nix
# /etc/nix/nix.conf or NixOS host module
nix.settings = {
  substituters = [
    "https://cache.nixos.org"
    "https://cache.internal.corp"
  ];
  trusted-public-keys = [
    "cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY="
    "downstream-cache-1:wK7b3...="
  ];
};
```

### SLSA Level 3 Build Provenance

For compliance with the Supply-chain Levels for Software Artifacts (SLSA) framework, modern pipelines attach cryptographic attestations generated during build execution:

1. **Hermetic Sandbox Enforcement:** The build runs with network access disabled (`sandbox = true`), ensuring no undeclared dependencies or external payloads can be pulled during compilation.
2. **Deterministic Inputs:** The `.drv` file records the exact cryptographic hashes of all input source archives, compilers, and dependencies.
3. **Attestation Signing (Sigstore / Cosign):** Build workers sign an in-toto attestation linking the input derivation hash to the output NAR hash, publishing the attestation to a transparency log.

---

## Checkpoint & Self-Review

When triaging vulnerabilities, applying backports, and publishing supply-chain metadata:

- [ ] Security patches are isolated, reviewed for clean context, and tested against targeted test suites.
- [ ] Backports preserve library ABI and Soname compatibility to avoid runtime linker crashes.
- [ ] Rebuild blast radius is analyzed before promoting global package set overrides.
- [ ] Ground-truth SBOMs (SPDX/CycloneDX) are generated directly from the cryptographic closure graph.
- [ ] VEX statements document non-exploitable vulnerabilities with explicit technical justifications.
- [ ] Binary caches enforce strict asymmetric key signing with trusted public key verification.
- [ ] Sandboxed builds satisfy SLSA hermeticity and deterministic input provenance.
