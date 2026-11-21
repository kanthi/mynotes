# Diffing NixOS configurations between revisions

**Goal:** Prove that a refactor changed only what you meant to change, by comparing the derivations of two configurations with `nix-diff`, between Git or Jujutsu revisions and against the live machine.

> [!NOTE]
> Based on Will Richardson's post [Comparing Nix Derivations in a JJ Repo](https://willhbr.net/2026/11/20/nix-jj-diff/) (20 November 2026). The `jj nix-diff` command is his own script; the commands below it are plain Nix. The sketch of the script is ours.

## Why this chapter exists

[Read an update before switching](15-read-an-update-before-switching.md) compares *closures* with `diff-closures`. That tells you which packages and sizes changed. It does not tell you that a refactor silently dropped an SSH authorised key or an option. A derivation diff does, because configuration files are inputs to the system derivation.

The scenario: a home server joins an existing config, and the two hosts' common parts are merged into shared modules. Nothing will fail to build if a key disappears, so you need a diff.

## Two levels of checking

| Check | Command | Good for |
|-------|---------|----------|
| Identical result? | Build both and compare store paths | A pure refactor that must change nothing |
| What differs? | `nix-diff` on the two `.drv` paths | A change that should differ only in the lines you edited |

Equal store paths mean identical outputs. When they differ, `nix-diff` gives a (large) structured diff of inputs, environment and arguments, so a removed authorised key shows up as a changed file.

## Getting derivation paths

The current config:

```bash
nix eval --raw '.#nixosConfigurations.yak.config.system.build.toplevel.drvPath'
```

The derivation behind the **running** machine (compare before you deploy):

```bash
ssh root@yak "nix-store -q --deriver /run/current-system"
```

This works when you built the system locally, since then both derivations exist in your local `/nix/store`. If the machine was built elsewhere, copy the derivation first or compare closures instead.

A specific Git commit, without checking it out:

```bash
nix eval --raw \
  "git+file://$REPO_PATH?rev=$COMMIT&dir=$SUBDIR#nixosConfigurations.$HOST.config.system.build.toplevel.drvPath"
```

> [!WARNING]
> Quote the URL. The `?` and `&` are shell metacharacters. The commit must be committed, because `git+file` with `rev=` reads from history, not from the working tree.

## Putting it together

```bash
old=$(nix eval --raw "git+file://$PWD?rev=$(git rev-parse HEAD~1)#nixosConfigurations.yak.config.system.build.toplevel.drvPath")
new=$(nix eval --raw "git+file://$PWD?rev=$(git rev-parse HEAD)#nixosConfigurations.yak.config.system.build.toplevel.drvPath")
nix run nixpkgs#nix-diff -- "$old" "$new" | less -R
```

With Jujutsu the same idea runs between `@-` and `@`, resolving revisions with `jj log -r REV --no-graph -T commit_id`. The post wraps this in a `jj nix-diff HOST` alias, which compares the working copy with its parent and accepts a **revset** to compare a chain of commits. See the Jujutsu primer in the VCS book of this library.

## Reading the output

- Expect a huge diff for any real change: the system derivation includes generated units, `/etc` files and activation scripts.
- Search it for the things you care about: `authorized_keys`, `users`, `allowedTCPPorts`, `.service`.
- An empty diff, or two identical `.drv` paths, is the strongest result a refactor can produce.

The author suggests a better tool would highlight categories (systemd services, open ports, users) instead of a generic diff. Until then, pair the diff with targeted evaluation:

```bash
nix eval .#nixosConfigurations.yak.config.users.users.alice.openssh.authorizedKeys.keys
nix eval .#nixosConfigurations.yak.config.networking.firewall.allowedTCPPorts
```

## Workflow for a refactor

1. Commit the starting point.
2. Make one refactoring step; commit.
3. Run the revision diff. For a pure refactor it should be empty; otherwise every difference must match your intent.
4. Before deploying, diff the new commit against the live machine's deriver.
5. Deploy ([deploy-rs](03-deploy-rs.md), [Colmena](04-colmena.md)), with rollback ready.

## Pitfalls

| Pitfall | Fix |
|---------|-----|
| Uncommitted changes missing from the diff | Commit first, or use a Jujutsu working-copy commit |
| `nix-store -q --deriver` returns nothing | The path was substituted from a cache and its deriver is unknown; compare with `diff-closures` instead |
| Different `nixpkgs` locks in the two revisions | The diff will show everything; keep `flake.lock` constant while refactoring |
| Treating "builds" as "correct" | Builds only prove syntax and types |
