# NixCon 2026: Nix core, builds and stores

**Goal:** Summarise the newest NixCon (Kraków, 25-28 September 2026) talks about the Nix tool itself: scaling, stores, builds, sandboxing, the module system and developer shells.

> [!NOTE]
> Source: the public [NixCon 2026 schedule](https://talks.nixcon.org/nixcon-2026/schedule/), 62 sessions. These notes use the **published abstracts**, not the recordings. Several projects are explicitly experimental. Earlier coverage: [NixCon 2025 field guide](05-nixcon-2025-field-guide.md). Operations and security talks are in [NixCon 2026: operations, security and platforms](11-nixcon-2026-operations-security-and-platforms.md).

## Scaling Nix

| Talk | Idea |
|------|------|
| **Scaling Nix** (Eelco Dolstra) | Nix is used far beyond its original "moderately sized Nixpkgs": giant monorepos and huge derivation graphs give long evaluation and CI times. The talk surveys ongoing and finished work to make Nix scale, in the spirit of "Nix is getting faster more slowly than Nixpkgs is getting slower" |
| **Nixception** (Guillaume Maudoux) | Incremental builds **without giving up the sandbox**. It speaks the Remote Build Execution (RBE) protocol on one side and the Nix daemon protocol on the other, turning each request into a normal derivation. It relies on the rarely used `recursive-nix` feature so the same caches serve builds inside and outside sandboxes |
| **Trampolining Nix** and **The thunk trap** (Mika Bohinen) | Writing an interpreter in Nix hits stack and laziness limits. `builtins.foldl'` is strict, yet leaves additions suspended, so reading the result can overflow the stack. Fix by computing the value before returning the next record; know `seq` versus `deepSeq` |
| **Biting your own tail off** (winterqt) | A company that uses Nix for everything finds a Nix weak spot in one of its own abstractions as it scales |

Practical lesson for your own flakes: force accumulators inside folds, avoid building large lazy chains, and measure evaluation with `NIX_SHOW_STATS=1`. See [Build optimization tuning](../13-platform-engineering/04-build-optimization-tuning.md).

## Serving the store at scale

Three talks attack the same problem: a swarm of ephemeral machines (CI jobs, spot instances, cloud dev environments, many campuses) that each need only a small part of a closure.

| Talk | Approach |
|------|----------|
| **Lazy Nix Store** (Srini, Zhaofeng Li) | A dynamic file system presenting `/nix/store` from an S3 bucket of deduplicated, content-addressed chunks. Because store paths are immutable, metadata for a whole NAR is fetched in one round trip and later reads are served from memory. Avoids re-downloading whole NAR closures and NFS "stat storms" |
| **CVMFS and mergerfs** (Cyriaque Millot) | CVMFS exposes the store as a read-only FUSE filesystem fetching file contents on demand from HTTP object storage. mergerfs layers a local read-write volume over it, so the Nix daemon sees a normal store and nothing needs patching |
| **DeltaNAR** (space-efficient deployments for aerospace) | Delta transfers between binary cache objects; the talk also shares learnings for future cache transfer protocols |

Compare with [Binary caches](../10-cicd/01-binary-caches.md): a cache copies whole closures, a lazy store copies only the files that are read.

## Sandboxing and trust

- **Unprivileged Privileged Nix** (Artemis Tosini): the Nix daemon is a highly privileged process that parses complex formats and runs code for untrusted users. The talk explores building as an **unprivileged user**, a rabbit hole of syscalls, systemd and shadow.
- **laut** (Martin Schwaighofer): follows his 2024 talk on rebuilding builders instead of trusting trust. Nix signatures say who signed, not who built. laut is a secondary binary shipped alongside Nix, aimed at attributing build results so trust is distributed instead of passed along like a game of telephone.
- **Build It All Yourself (And Prove It)** (Kierán Meinhardt): developers work unrestricted while releases are **rebuilt from scratch in an air-gapped environment with Hydra**, so supply-chain assurance does not slow development. See [Remote builders](../10-cicd/07-remote-builders.md).

## Language, modules and the ecosystem

| Talk | Idea |
|------|------|
| **Statix** (bme) | Rework of the module system in Racket that separates values from options, giving cheap, correct diagnosis of most errors (no more clueless infinite recursion) and a hole-driven REPL workflow |
| **Why I changed my mind about flakes** (Valentin Gagarin) | Argues the long-running controversy, with flakes still experimental after seven years, was never really about flakes but about a failure of communication |
| **Adding to the pile** (Yannik Sander) | The architecture behind package sets: `mkScope`, `mkExtensible`, overlays, and where automation fails. See [Agent sandboxes and overlays](../13-platform-engineering/12-agent-sandboxes-loop-engineering-and-overlays.md) |
| **Mainstream Nix** (Joachim Schiele) | libnix: a Nix backend for cargo and a native Windows port |
| **Nix store is my favorite database** (Anuj More) | Models graphs (flight networks, chess positions) as store derivations; good practice for `buildInputs` versus runtime references |

## Developer shells and build systems

- **The new `.devShell` interface for `nix develop`** (Lisanna Dettwyler): `nix develop` was rewritten to support a `.devShell` interface provided by Nixpkgs, with documented extension points for custom dev shells. Re-test existing shells after upgrading.
- **The ministry of silly builds** (Aleksander Gondek, Artur Stachecki): a Bazel 9 derivation builder. The existing `buildBazelPackage` has to vendor all dependencies up front because Bazel fetches lazily, which clashes with Nix declaring everything in advance.
- **Declarative Hardware Architectures using Nix** (Tristan Ross): one unified toolchain definition for FPGA and ASIC design.

## Checklist

- [ ] Evaluation time of your flake measured, with a budget
- [ ] Ephemeral CI machines fetch only what they use (substituter proximity or lazy store)
- [ ] Release artefacts rebuilt in a clean environment before shipping
- [ ] Dev shells re-tested against the new `nix develop` interface
