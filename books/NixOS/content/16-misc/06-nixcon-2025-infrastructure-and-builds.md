# NixCon 2025: Nix infrastructure and build internals

**Goal:** Understand how the Nix project's own infrastructure works and which build-layer experiments from NixCon 2025 are worth tracking.

> [!NOTE]
> Based on the published talk abstracts. Experimental tools below are early-stage; check their repositories before use.

## How a change reaches `/nix/store`

"How NixOS is built: from Pull Request to your /nix/store" follows a nixpkgs change through every CI system, how they are defined in the codebases, and the security implications of each step. Pair it with [How a release is built](02-how-a-release-is-built.md).

```text
 PR in nixpkgs -> CI checks -> merge -> Hydra evaluates -> builders build
      -> outputs signed and uploaded -> cache.nixos.org -> your substituter
```

## cache.nixos.org

The fireside chat "The Nix Binary Cache and AWS" described:
- The cache handles thousands of requests per second, nearly **6 billion requests per month**.
- AWS and the Nix Foundation have a partnership that keeps it available.
- Its **simple ("dumb") design is deliberate**: static objects keyed by store path hash. That is why it scales and why you can run your own.

Related talks:

| Talk | Idea |
|------|------|
| Building a Nix Binary Cache for Fun and Profit | Lessons from implementing a cache server from scratch |
| Styx | Deduplicates a Nix store on disk and on the wire, adds differential compression and on-demand fetching, aimed at faster NixOS updates with less disk |
| Share your daemons with your peers | Forwarding a remote Nix daemon over **P2P** instead of `ssh-ng`, which is the only built-in active remote store |

See [Binary caches](../10-cicd/01-binary-caches.md) for running your own cache.

## Hydra, rewritten

Simon Hauser's talk covers the rewrite of the Hydra **Queue Runner** in Rust.

| Before | After |
|--------|-------|
| SSH to builders; talks directly to the running Nix daemon | **gRPC** protocol |
| Hard limit on connected builders | Substantially more concurrent machines |
| Limited tracing | Comprehensive tracing |
| One scheduling view | A separate queue per platform; builder utilisation monitoring informs scheduling |

Components named in the talk: evaluator, Queue Runner, PostgreSQL, hydra-notify, hydra-web.

## Sandboxing and the Nix library

John Ericson's **DerivationBuilder** work extracts the sandboxing logic from the build scheduler into its own component with a simple FFI interface, so other programs (not only Nix) can build derivations in the same sandbox. Why it matters: sandbox mistakes are silent until the wrong derivation arrives.

## Evaluation and build speed

| Talk | Result |
|------|--------|
| Embarrassingly parallel evaluations | 300+ packages evaluated for a Bazel monorepo: **5 minutes down to 5 seconds** using several eval caches, flakes, hand-crafted store paths and parallel evaluation; two PRs to Nix |
| Nix Ninja | Uses the experimental **dynamic derivations** to build at compilation-unit granularity from ninja files (CMake, meson). Can compile CppNix, still early |
| garn | Define derivations in TypeScript; v2 aims to use Nix only to build `.drv` files |

> [!TIP]
> For your own fleet, the cheap lessons are: cache evaluation results, split independent evaluations, and avoid import-from-derivation in hot paths. See [Build optimization tuning](../13-platform-engineering/04-build-optimization-tuning.md).

## Formatting

"The bikes have been shed" tells how **nixfmt** became official: Nix had no formatter for 15 years, the standardisation RFC took **3 years, 50 meetings and 600 comments**, and about 40k Nix files with 15k monthly commits were reformatted. Run `nixfmt` in CI so formatting never appears in review.

## Ports and research

- **NixOS on LoongArch64:** state of the ecosystem and issues met porting nixpkgs/NixOS.
- **Graph2Diff for nixpkgs (thesis progress report):** harvest failure/fix PR pairs plus ASTs and build logs, train a graph neural network to propose edits; chosen over LLM repair partly for cost at nixpkgs scale.
- **Deterministic logos with Nix:** generate logo SVGs as fixed-output derivations; lesson: expose **overlays and NixOS modules** rather than making consumers hack flake outputs.
