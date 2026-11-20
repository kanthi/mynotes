---
title: 'Jujutsu (jj) Primer: Revsets, DAG Operations, Bookmarks, and Stacked PRs'
---

# Jujutsu (jj) Primer: Revsets, DAG Operations, Bookmarks, and Stacked PRs

Once the working-copy-as-commit (`@`) mental model clicks, day-to-day Jujutsu work shifts from staging files to sculpting a commit graph. Instead of juggling local feature branches and interactive rebases that halt on every conflict, `jj` treats commits as malleable nodes in an anonymous directed acyclic graph (DAG): descendant commits rebase automatically when you edit an ancestor, conflicts are stored inside commits as first-class objects until you are ready to resolve them, and bookmarks act as explicit movable labels for Git remotes.

This primer walks through a complete end-to-end `jj` workflow on a colocated Git repository—from configuring scopes and querying the graph with revsets to navigating stacks, resolving deferred conflicts via the squash workflow, reshaping history (`rebase`, `squash`, `abandon`, `split`), driving Git remotes through bookmarks, and maintaining stacked pull requests.

## Learning goals

By the end of this chapter you can:

- Configure `jj` across `--user`, `--repo`, and `--workspace` scopes and inspect commits via unambiguous change-ID and commit-ID prefixes
- Evolve changes in place or advance the graph with `jj desc`, `jj new`, and `jj commit`, including branching anonymously without named branches
- Write revset expressions (`@-`, `x++`, `x::y`, `x::`, boolean operators, and functions like `author()`, `trunk()`, `mine()`) to select arbitrary commit subgraphs
- Navigate up and down a commit stack with `jj edit`, `jj prev`, and `jj next`, relying on automatic descendant rebasing
- Resolve first-class conflicts directly in place or cleanly in isolation via the squash workflow (`jj new <conflicted>` → resolve → `jj squash`), backed by `jj undo` and `jj op log`
- Reshape the DAG with `jj rebase`, `jj squash --from/--into`, `jj abandon`, `jj diffedit`, and `jj split`
- Bridge branchless local graphs to Git remotes with generated bookmarks (`jj git push -c`), the bookmark squash flow, the bookmark advance flow (`jj bookmark advance`), and trunk rebasing

## Why Jujutsu composes cleanly

`jj` offers a small set of orthogonal primitives that compose without sharp edges:

1. **No index / staging area:** Your working copy is always a real commit (`@`). Editing files updates `@` automatically on the next `jj` command.
2. **Automatic descendant rebasing:** Editing an older commit in the middle of a stack automatically rebases every descendant on top of your changes.
3. **First-class conflicts:** If an automatic rebase produces a conflict, `jj` does not abort or trap your terminal in a mid-rebase state. It records the algebraic conflict inside the affected commit node and lets you keep working until you choose to resolve it.
4. **Universal undo via the operation log:** Every action is recorded in an append-only operation log (`jj op log`), making `jj undo` deterministic across history rewrites.
5. **Colocated Git compatibility:** Running `jj git init` inside an existing Git repository lets `jj` and `git` operate side-by-side on the same working tree, so you can adopt `jj` locally without disrupting teammates or CI pipelines.

## Installation and configuration scopes

On Linux (and macOS), `jj` installs cleanly from system package managers or Cargo:

```bash
# Arch Linux
pacman -S jujutsu

# NixOS / Nix
nix-shell -p jujutsu

# From source via Cargo (requires a recent Rust toolchain)
cargo install --locked --bin jj jj-cli

# macOS (Homebrew)
brew install jj
```

To enable `jj` inside an existing Git repository, initialize it in colocated mode:

```bash
jj git init
```

From that point onward, `jj` and `git` automatically synchronize repository state on every command.

### Configuration scopes

Set your author identity globally in the `--user` configuration scope:

```bash
jj config set --user user.name "Maya Lin"
jj config set --user user.email "maya@example.com"
```

You can open the TOML configuration file directly in your `$EDITOR`:

```bash
jj config edit --user
```

Beyond `--user`, `jj config` supports `--repo` (repository-specific settings) and `--workspace` (workspace-specific overrides). Inspect the active merged configuration at any time with:

```bash
jj config list
```

## Commits, change IDs, and shortest unique prefixes

A **commit** (or **revision**) in `jj` is a snapshot of the repository descending from one or more parent commits—identical in structure to a Git commit. When backed by Git storage, a `jj` **commit ID** *is* the hexadecimal Git commit SHA.

Run `jj log` in a freshly colocated repository:

```text
$ jj log
@  tzvpxowk maya@example.com 2026-11-15 12:17:12 914685ac
│  (empty) (no description set)
◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
│  Tune rate limiter defaults
~
```

Every line of `jj log` shows two identifiers:

- **Change ID (left, e.g., `mkrwlvpn` and `tzvpxowk`):** A stable identifier using letters `k` through `z` that tracks a *logical change* across rewrites, amends, and rebases.
- **Commit ID (right, e.g., `4cfa7b1e` and `914685ac`):** The immutable content-addressed Git hash (`0`–`9`, `a`–`f`), which changes whenever file contents, parent pointers, timestamps, or descriptions change.

> [!TIP]
> In terminal output, `jj` highlights the **shortest unique prefix** of every change ID and commit ID (for instance, bolding `t` in `tzvpxowk` or `4` in `4cfa7b1e`). You never need to copy-paste full hashes—passing the single highlighted letter or a 2-character prefix is enough for any `jj` command.

Inspect commit `4cfa7b1e` using its shortest unique prefix `4`:

```text
$ jj show 4
Commit ID: 4cfa7b1e8291b40ca7ee63d1ce48b11adca8f9ea
Change ID: mkrwlvpntrkxqppotnrvutxlvrvqyxmy
Bookmarks: main main@git main@origin
Author   : Maya Lin <maya@example.com> (2026-11-15 12:10:46)
Committer: Maya Lin <maya@example.com> (2026-11-15 12:10:46)

    Tune rate limiter defaults

diff --git a/settings.py b/settings.py
index 7366a4e31d..174ffea06b 100644
--- a/settings.py
+++ b/settings.py
@@ -1,1 +1,1 @@
-MAX_BATCH_SIZE = 25
+MAX_BATCH_SIZE = 50
```

If you prefer syntax-highlighted side-by-side diffs, `jj` integrates cleanly with pagers and diff formatters such as [Delta](https://github.com/dandavison/delta) via the `ui.pager` and `ui.diff-formatter` TOML settings.

## Creating and evolving changes

In the `jj log` output above, the `@` symbol marks the **working-copy commit** (`914685ac`, change ID `tzvpxowk`). The diamond `◆` marks an **immutable commit** (such as `main` pushed to a remote), and `○` marks mutable ancestor or descendant commits.

When `@` is freshly created on top of `main`, three core inspection commands show its clean state:

1. `jj show` prints the metadata and diff of `@` (currently empty):

   ```text
   $ jj show
   Commit ID: 914685aca19639dbd6ccb53e8c1340e78c36d59a
   Change ID: tzvpxowkksrltlrkyqukqqsxpltznztk
   Author   : Maya Lin <maya@example.com> (2026-11-15 12:17:12)
   Committer: Maya Lin <maya@example.com> (2026-11-15 12:17:12)

       (no description set)
   ```

2. `jj status` summarizes the working copy (`@`) and its parent (`@-`):

   ```text
   $ jj status
   The working copy has no changes.
   Working copy  (@) : tzvpxowk 914685ac (empty) (no description set)
   Parent commit (@-): mkrwlvpn 4cfa7b1e main | Tune rate limiter defaults
   ```

3. `jj diff` prints the working-copy delta against `@-` (empty when no files have been modified):

   ```text
   $ jj diff
   ```

### Automatic snapshotting on file edit

Now create a new file `worker.py` in the working directory and run `jj show`:

```text
$ jj show
Commit ID: 73c81d40c534705cfb753c722222290e9185b22c
Change ID: tzvpxowkksrltlrkyqukqqsxpltznztk
Author   : Maya Lin <maya@example.com> (2026-11-15 12:26:43)
Committer: Maya Lin <maya@example.com> (2026-11-15 12:26:43)

    (no description set)

diff --git a/worker.py b/worker.py
new file mode 100644
index 0000000000..c74cd2e0e0
--- /dev/null
+++ b/worker.py
@@ -0,0 +1,2 @@
+def sync_records():
+  pass
```

Check the commit graph again:

```text
$ jj log
@  tzvpxowk maya@example.com 2026-11-15 12:26:43 73c81d40
│  (no description set)
◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
│  Tune rate limiter defaults
~
```

Notice what happened automatically:

- There was no `git add`. Running `jj show` snapshotted the working directory into `@`.
- The `(empty)` marker disappeared from `@`.
- The **commit ID** changed from `914685ac` to `73c81d40` because the tree contents changed.
- The **change ID** remained `tzvpxowk`.

### Describing a change (`jj desc`)

Attach a description (commit message) to `@` using `jj desc` (which opens your editor) or inline with `-m`:

```text
$ jj desc -m "Add record sync worker"
Working copy  (@) now at: tzvpxowk 8d41cca9 Add record sync worker
Parent commit (@-)      : mkrwlvpn 4cfa7b1e main | Tune rate limiter defaults
```

Because the commit message is hashed into the Git commit object, the commit ID rotated again (`8d41cca9`), while the change ID stayed `tzvpxowk`.

### Advancing to a new change (`jj new` and `jj commit`)

Once `tzvpxowk` is ready and you want to build the caller in a separate commit on top of it, run `jj new`:

```text
$ jj new
Working copy  (@) now at:plrtzxvn 5bba3cee (empty) (no description set)
Parent commit (@-)      : tzvpxowk 8d41cca9 Add record sync worker
```

```text
$ jj log
@  plrtzxvn maya@example.com 2026-11-15 12:35:49 5bba3cee
│  (empty) (no description set)
○  tzvpxowk maya@example.com 2026-11-15 12:33:23 8d41cca9
│  Add record sync worker
◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
│  Tune rate limiter defaults
~
```

`jj new` creates a fresh empty change (`plrtzxvn`) as a child of `tzvpxowk` and moves `@` there.

If you prefer the traditional "describe what I just edited and immediately move to a new empty working copy" cadence, `jj commit` combines `jj desc` + `jj new` into one command. Edit `worker.py` to invoke `sync_records()` and run `jj commit`:

```text
$ jj commit -m "Run sync worker in pipeline"
Working copy  (@) now at: vqwyusnt 21098bf1 (empty) (no description set)
Parent commit (@-)      : plrtzxvn f2c17789 Run sync worker in pipeline
```

```text
$ jj log
@  vqwyusnt maya@example.com 2026-11-15 12:41:30 21098bf1
│  (empty) (no description set)
○  plrtzxvn maya@example.com 2026-11-15 12:41:30 f2c17789
│  Run sync worker in pipeline
○  tzvpxowk maya@example.com 2026-11-15 12:33:23 8d41cca9
│  Add record sync worker
◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
│  Tune rate limiter defaults
~
```

Because `@` has moved onto the new empty change `vqwyusnt`, running a bare `jj show` now inspects `vqwyusnt`. To inspect the change we just committed (`plrtzxvn`), pass its unique prefix `p`:

```text
$ jj show p
Commit ID: f2c177892aa1a565c1e5d38ef27a308aa58e7e4c
Change ID: plrtzxvnuuskxprvtuznquqzqzzwvnzk
Author   : Maya Lin <maya@example.com> (2026-11-15 12:41:30)
Committer: Maya Lin <maya@example.com> (2026-11-15 12:41:30)

    Run sync worker in pipeline

diff --git a/worker.py b/worker.py
index c74cd2e0e0..0714ec450b 100644
--- a/worker.py
+++ b/worker.py
@@ -1,2 +1,4 @@ def sync_records():
 def sync_records():
   pass
+
+dispatch_job(sync_records())
```

### Branchless graph branching and recording intent

Because `jj` decouples commits from named branches, you can create an empty commit with a description purely to record intent—and you can branch directly off any earlier change ID without naming a Git branch.

Suppose we want to start a documentation commit directly on top of `tzvpxowk` (prefix `t`), parallel to `plrtzxvn`:

```text
$ jj new t -m "Document record sync worker"
Working copy  (@) now at: yusntzqw 38b4eeca (empty) Document record sync worker
Parent commit (@-)      : tzvpxowk 8d41cca9 Add record sync worker
Added 0 files, modified 1 files, removed 0 files
```

Look at the resulting DAG:

```text
$ jj log
@  yusntzqw maya@example.com 2026-11-15 12:46:35 38b4eeca
│  (empty) Document record sync worker
│ ○  plrtzxvn maya@example.com 2026-11-15 12:41:30 f2c17789
├─┘  Run sync worker in pipeline
○  tzvpxowk maya@example.com 2026-11-15 12:33:23 8d41cca9
│  Add record sync worker
◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
│  Tune rate limiter defaults
~
```

Notice that `vqwyusnt` (the previous empty `@` with no description) was automatically garbage-collected when we moved `@` away from it, whereas `yusntzqw` is preserved because it has a description. More importantly, we now have a two-arm fork above `tzvpxowk` without ever running `git checkout -b`.

## Revsets: querying the commit graph

A **revset** (a functional query language inspired by Mercurial) selects a set of revisions from the DAG. Almost every `jj` command accepts revsets. Even a single prefix like `t` is a revset resolving to `tzvpxowk`. You can test any revset expression safely with `jj log -r '<REVSET>'`.

| Revset expression | Meaning |
|---|---|
| `@` | The working-copy commit |
| `@-` | Parent(s) of `@` |
| `x+` / `x++` | Children of `x` / grandchildren of `x` |
| `x::y` | All commits on the DAG path between `x` and `y` (inclusive) |
| `x::` | Commit `x` and all of its descendants |
| `::x` | Commit `x` and all of its ancestors |
| `A \| B` | Union of revsets `A` and `B` |
| `A & B` | Intersection of revsets `A` and `B` |
| `~A` | Complement (all visible commits not in `A`) |
| `author("pattern")` | Commits whose author matches `pattern` |
| `trunk()` | The head of the primary trunk bookmark (`main` / `master`) |
| `mine()` | Commits authored by the current configured user |
| `mutable()` | Commits not marked immutable |

### Revset examples in practice

1. **Parent of the working copy (`@-`):**

   ```text
   $ jj log -r @-
   ○  tzvpxowk maya@example.com 2026-11-15 12:33:23 8d41cca9
   │  Add record sync worker
   ~
   ```

2. **Grandchildren of `m` (`m++`):**

   ```text
   $ jj log -r m++
   @  yusntzqw maya@example.com 2026-11-15 12:51:45 6c77182b
   │  Document record sync worker
   ~

   ○  plrtzxvn maya@example.com 2026-11-15 12:41:30 f2c17789
   │  Run sync worker in pipeline
   ~
   ```

3. **Ancestry range from `m` to `@` inclusive (`m::@`):**

   ```text
   $ jj log -r m::@
   @  yusntzqw maya@example.com 2026-11-15 12:51:45 6c77182b
   │  Document record sync worker
   ○  tzvpxowk maya@example.com 2026-11-15 12:33:23 8d41cca9
   │  Add record sync worker
   ◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
   │  Tune rate limiter defaults
   ~
   ```

4. **Subgraph of `t` and all its descendants (`t::`):**

   ```text
   $ jj log -r t::
   @  yusntzqw maya@example.com 2026-11-15 12:51:45 6c77182b
   │  Document record sync worker
   │ ○  plrtzxvn maya@example.com 2026-11-15 12:41:30 f2c17789
   ├─┘  Run sync worker in pipeline
   ○  tzvpxowk maya@example.com 2026-11-15 12:33:23 8d41cca9
   │  Add record sync worker
   ~
   ```

5. **Combining functions and boolean operators (`&` and `|`):**

   ```text
   $ jj log -r 'author("maya@example.com") & (t | y)'
   @  yusntzqw maya@example.com 2026-11-15 12:51:45 6c77182b
   │  Document record sync worker
   ○  tzvpxowk maya@example.com 2026-11-15 12:33:23 8d41cca9
   │  Add record sync worker
   ~
   ```

## Navigating the commit graph and automatic rebasing

Suppose that while writing documentation in `yusntzqw`, we realize `sync_records()` in parent change `tzvpxowk` has a bug. In Git, switching back to edit an earlier commit requires stashing or committing WIP and starting an interactive rebase. In `jj`, you simply move `@` directly onto `tzvpxowk` with `jj edit`:

```text
$ jj edit t
Working copy  (@) now at: tzvpxowk 8d41cca9 Add record sync worker
Parent commit (@-)      : mkrwlvpn 4cfa7b1e main | Tune rate limiter defaults
Added 0 files, modified 0 files, removed 1 files
```

> [!NOTE]
> Instead of `jj edit t`, you can also run `jj edit @-` (since `@-` is evaluated before moving `@`), or step vertically through a linear stack with `jj prev --edit` and `jj next --edit`.

Verify that `@` is now at `tzvpxowk` while both child changes (`yusntzqw` and `plrtzxvn`) remain intact above it:

```text
$ jj log
○  yusntzqw maya@example.com 2026-11-15 12:51:45 6c77182b
│  Document record sync worker
│ ○  plrtzxvn maya@example.com 2026-11-15 12:41:30 f2c17789
├─┘  Run sync worker in pipeline
@  tzvpxowk maya@example.com 2026-11-15 12:33:23 8d41cca9
│  Add record sync worker
◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
│  Tune rate limiter defaults
~
```

### Automatic descendant rebasing

Now edit `worker.py` while `@` is on `tzvpxowk` and run `jj log`:

```text
$ jj log
Rebased 2 descendant commits onto updated working copy.
○  yusntzqw maya@example.com 2026-11-15 13:03:41 8a391d04
│  Document record sync worker
│ ○  plrtzxvn maya@example.com 2026-11-15 13:03:41 52b64c91
├─┘  Run sync worker in pipeline
@  tzvpxowk maya@example.com 2026-11-15 13:03:41 e8a46920
│  Add record sync worker
◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
│  Tune rate limiter defaults
~
```

`jj` snapshotted `tzvpxowk` (giving it new commit ID `e8a46920`) and **automatically rebased** its two descendants (`plrtzxvn` and `yusntzqw`), updating their commit IDs while preserving all three change IDs.

## Handling conflicts as first-class state

What if our edit to `tzvpxowk` modifies the exact lines in `worker.py` that `plrtzxvn` touched? In Git, `git rebase` stops midway and blocks all other work until you resolve the conflict or `git rebase --abort`.

In `jj`, automatic rebasing **never fails**. It records the conflict inside `plrtzxvn`, marks the node with `×` and `(conflict)`, and lets you keep working:

```text
$ jj log
Rebased 2 descendant commits onto updated working copy.
○  yusntzqw maya@example.com 2026-11-15 13:05:35 c85e9ea2
│  Document record sync worker
│ ×  plrtzxvn maya@example.com 2026-11-15 13:05:35 f673caa1 (conflict)
├─┘  Run sync worker in pipeline
@  tzvpxowk maya@example.com 2026-11-15 13:05:35 d612b0ce
│  Add record sync worker
◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
│  Tune rate limiter defaults
~
```

When you are ready to fix the conflict, one approach is to edit `plrtzxvn` directly:

```text
$ jj edit p
Working copy  (@) now at: plrtzxvn f673caa1 (conflict) Run sync worker in pipeline
Parent commit (@-)      : tzvpxowk d612b0ce Add record sync worker
Added 0 files, modified 1 files, removed 0 files
Warning: There are unresolved conflicts at these paths:
worker.py    2-sided conflict
```

When `@` is a conflicted commit, `jj` materializes diff-style conflict markers into the working file `worker.py`:

```text
def sync_records():
  print("Starting record sync...")
<<<<<<< conflict 1 of 1
%%%%%%% diff from: tzvpxowk e8a46920 "Add record sync worker" (parents of rebased revision)
\\\\\\\        to: tzvpxowk d612b0ce "Add record sync worker" (rebase destination)
-  pass
+  return 100
+++++++ plrtzxvn 52b64c91 "Run sync worker in pipeline" (rebased revision)
  pass

dispatch_job(sync_records())
>>>>>>> conflict 1 of 1 ends
```

Notice how readable `jj`'s conflict format is: the `%%%%%%%` section shows the exact diff between the old parent and new parent, and the `+++++++` section shows the snapshot from the rebased revision.

Once you remove the conflict markers in your editor and save `worker.py`, the next `jj` command snapshots the resolution and clears the `(conflict)` state automatically:

```text
$ jj status
Working copy changes:
M worker.py
Working copy  (@) : plrtzxvn 45e3f1b2 Run sync worker in pipeline
Parent commit (@-): tzvpxowk d612b0ce Add record sync worker
```

## Undo and the operation log (`jj undo` and `jj op log`)

Suppose you resolved that conflict incorrectly and want to rewind. In `jj`, every command is logged as an atomic transaction in the **operation log**. Running `jj undo` reverts the most recent operation:

```text
$ jj undo
Undid operation: 48cfc1852a28 (2026-11-15 13:08:39) snapshot working copy
Restored to operation: a245db098c60 (2026-11-15 13:07:18) edit commit f673caa1876f3ea76248c3f730e9e74d3e28e8a0
Working copy  (@) now at: plrtzxvn f673caa1 (conflict) Run sync worker in pipeline
Parent commit (@-)      : tzvpxowk d612b0ce Add record sync worker
Added 0 files, modified 1 files, removed 0 files
Warning: There are unresolved conflicts at these paths:
worker.py    2-sided conflict
New conflicts appeared in 1 commits:
  plrtzxvn f673caa1 (conflict) Run sync worker in pipeline
Hint: To resolve the conflicts, start by creating a commit on top of
the conflicted commit:
  jj new plrtzxvn
Then use `jj resolve`, or edit the conflict markers in the file directly.
Once the conflicts are resolved, you can inspect the result with `jj diff`.
Then run `jj squash` to move the resolution into the conflicted commit.
```

Beyond single-step `jj undo` (and `jj redo`), you can inspect the full history of repository operations with `jj op log`, restore the entire repository to an earlier point in time with `jj op restore <op-id>`, or surgically invert a single past operation with `jj op revert <op-id>`.

## The squash workflow

Look closely at the `Hint:` emitted by `jj` above. Editing a large commit directly with `jj edit` mixes your new edits with the commit's existing diff. The **squash workflow** keeps your incremental edits isolated in a child commit until you are satisfied, then folds them into the target commit:

1. Create a new empty working-copy commit on top of the conflicted commit `plrtzxvn`:

   ```text
   $ jj new p
   Working copy  (@) now at: xollozwq c989989b (conflict) (empty) (no description set)
   Parent commit (@-)      : plrtzxvn f673caa1 (conflict) Run sync worker in pipeline
   Warning: There are unresolved conflicts at these paths:
   worker.py    2-sided conflict
   ```

   Because `xollozwq` descends from a conflicted commit, it inherits the conflict markers in the working directory:

   ```text
   $ jj log -r p::
   @  xollozwq maya@example.com 2026-11-15 13:11:44 c989989b (conflict)
   │  (empty) (no description set)
   ×  plrtzxvn maya@example.com 2026-11-15 13:05:35 f673caa1 (conflict)
   │  Run sync worker in pipeline
   ~
   ```

2. Resolve the conflict markers in `worker.py` (or run `jj resolve` to invoke a 3-way merge tool) and check `jj log`:

   ```text
   $ jj log -r p::
   @  xollozwq maya@example.com 2026-11-15 13:14:12 a2292bfd
   │  (no description set)
   ×  plrtzxvn maya@example.com 2026-11-15 13:05:35 f673caa1 (conflict)
   │  Run sync worker in pipeline
   ~
   ```

   Now `@` (`xollozwq`) is conflict-free and contains *only* the conflict-resolution delta, which you can review in isolation with `jj diff`.

3. Fold the resolution down into `plrtzxvn` with `jj squash`:

   ```text
   $ jj squash
   Working copy  (@) now at: kywoywqv e5a659c4 (empty) (no description set)
   Parent commit (@-)      : plrtzxvn 8319efc8 Run sync worker in pipeline
   Existing conflicts were resolved or abandoned from 1 commits.
   ```

   ```text
   $ jj log -r p::
   @  kywoywqv maya@example.com 2026-11-15 13:15:39 e5a659c4
   │  (empty) (no description set)
   ○  plrtzxvn maya@example.com 2026-11-15 13:15:39 8319efc8
   │  Run sync worker in pipeline
   ~
   ```

## Reshaping the commit graph

With automatic rebasing and `jj undo` as a safety net, reshaping the DAG takes seconds. Start from our current two-branch graph:

```text
$ jj log
@  kywoywqv maya@example.com 2026-11-15 13:15:39 e5a659c4
│  (empty) (no description set)
○  plrtzxvn maya@example.com 2026-11-15 13:15:39 8319efc8
│  Run sync worker in pipeline
│ ○  yusntzqw maya@example.com 2026-11-15 13:05:35 c85e9ea2
├─┘  Document record sync worker
○  tzvpxowk maya@example.com 2026-11-15 13:05:35 d612b0ce
│  Add record sync worker
◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
│  Tune rate limiter defaults
~
```

### Linearizing and inserting commits with `jj rebase`

Suppose we want a single linear stack where documentation (`yusntzqw`) sits between implementation (`tzvpxowk`) and pipeline usage (`plrtzxvn`). Rebasing source `-s p` (which moves `plrtzxvn` and all its descendants) onto destination `-o y` linearizes the graph:

```text
$ jj rebase -s p -o y
Rebased 2 commits to destination.
Working copy  (@) now at: kywoywqv 1516671d (empty) (no description set)
Parent commit (@-)      : plrtzxvn a831346c Run sync worker in pipeline
Added 1 files, modified 0 files, removed 0 files
```

```text
$ jj log
@  kywoywqv maya@example.com 2026-11-15 13:20:18 1516671d
│  (empty) (no description set)
○  plrtzxvn maya@example.com 2026-11-15 13:20:18 a831346c
│  Run sync worker in pipeline
○  yusntzqw maya@example.com 2026-11-15 13:05:35 c85e9ea2
│  Document record sync worker
○  tzvpxowk maya@example.com 2026-11-15 13:05:35 d612b0ce
│  Add record sync worker
◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
│  Tune rate limiter defaults
~
```

> [!TIP]
> `jj rebase` offers three targeting modes:
> - `-s <source> -o <dest>`: move `<source>` and all its descendants onto `<dest>`
> - `-b <branch> -o <dest>`: move the whole branch/subgraph relative to destination
> - `-r <rev> --before <target>` / `--after <target>` / `-o <dest>`: pluck a single revision `<rev>` and splice it into a new position while automatically reconnecting its children to its old parent. For instance, `jj rebase -r y --before p` achieves the exact same linearization.

Similarly, to insert a preparatory commit *below* `tzvpxowk`, you can run `jj new --before t -m "Scaffold worker types"` directly, or create a commit on `m` and rebase `t` onto it:

```text
$ jj new m -m "Scaffold worker types"
Working copy  (@) now at: xzmtyolw cf6db3ac (empty) Scaffold worker types
Parent commit (@-)      : mkrwlvpn 4cfa7b1e main | Tune rate limiter defaults
Added 0 files, modified 0 files, removed 2 files

$ jj rebase -s t -o x
Rebased 3 commits to destination.
```

```text
$ jj log
○  plrtzxvn maya@example.com 2026-11-15 13:28:02 deb1ef99
│  Run sync worker in pipeline
○  yusntzqw maya@example.com 2026-11-15 13:28:02 fd0ef27a
│  Document record sync worker
○  tzvpxowk maya@example.com 2026-11-15 13:28:02 833f8379
│  Add record sync worker
@  xzmtyolw maya@example.com 2026-11-15 13:23:29 cf6db3ac
│  (empty) Scaffold worker types
◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
│  Tune rate limiter defaults
~
```

If you prefer visual interactive reordering akin to `git rebase -i`, run `jj arrange` to open the built-in TUI graph arranger.

### Combining arbitrary commits with `jj squash --from --into`

Without flags, `jj squash` folds `@` into `@-`. With `--from` and `--into`, it folds any revision into any other revision. To combine documentation (`y`) directly into implementation (`t`):

```text
$ jj squash --from y --into t
Rebased 1 descendant commits.
```

`jj` opens your editor to combine the two descriptions. Once saved:

```text
$ jj log
○  plrtzxvn maya@example.com 2026-11-15 13:31:30 dc579936
│  Run sync worker in pipeline
○  tzvpxowk maya@example.com 2026-11-15 13:31:30 ca4f5fe8
│  Add and document record sync worker
@  xzmtyolw maya@example.com 2026-11-15 13:23:29 cf6db3ac
│  (empty) Scaffold worker types
◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
│  Tune rate limiter defaults
~
```

### Dropping commits or hunks with `jj abandon` and `jj diffedit`

To remove a commit from the graph while automatically rebasing its children onto its parent, use `jj abandon`:

```text
$ jj abandon x
Abandoned 1 commits:
  xzmtyolw cf6db3ac (empty) Scaffold worker types
Rebased 2 descendant commits onto parents of abandoned commits.
Working copy  (@) now at: zwrrwyvx e571gce5 (empty) (no description set)
Parent commit (@-)      : mkrwlvpn 4cfa7b1e main | Tune rate limiter defaults
```

```text
$ jj log
@  zwrrwyvx maya@example.com 2026-11-15 13:33:49 e571gce5
│  (empty) (no description set)
│ ○  plrtzxvn maya@example.com 2026-11-15 13:33:49 c2395e72
│ │  Run sync worker in pipeline
│ ○  tzvpxowk maya@example.com 2026-11-15 13:33:49 ea04e1b7
├─┘  Add and document record sync worker
◆  mkrwlvpn maya@example.com 2026-11-15 12:10:46 main 4cfa7b1e
│  Tune rate limiter defaults
~
```

If you only want to discard specific files or hunks from a commit rather than dropping the whole revision, run `jj diffedit -r <rev>` to open an interactive diff editor.

### Splitting commits with `jj split`

To break a single commit into two sequential commits, run `jj split -r <rev>`:

```bash
jj split -r t
```

`jj split` launches an interactive diff selector where you choose which files or individual hunks stay in the first commit; the remaining changes move into a newly created child commit, and you are prompted for both commit descriptions. For scripted or agent-assisted workflows, tools like [jj-hunk](https://github.com/laulauland/jj-hunk) provide non-interactive hunk selection on top of `jj split`.

## Bookmarks and Git branches

`jj` is branchless at its core, but Git remotes require named branches (`refs/heads/*`). `jj` bridges this gap with **bookmarks**—movable named pointers attached to commits that map 1:1 to Git branches when pushing or fetching.

Unlike Git branches, **bookmarks do not automatically advance when you run `jj new` or `jj commit`**. A bookmark stays pinned to its change ID (and follows that change ID when amended or rebased) until you explicitly move or advance it.

### Pushing a new branch with auto-generated bookmarks

Check your configured Git remotes:

```text
$ jj git remote list
origin git@github.com:mayalin/sync-service.git
```

When you are ready to open a pull request for change `plrtzxvn` (`p`), you do not even have to invent a branch name. Passing `-c` (`--change`) to `jj git push` generates a bookmark named `push-<change-id>` automatically and pushes it to `origin`:

```text
$ jj git push -c p
Creating bookmark push-plrtzxvnuusk for revision plrtzxvn
Changes to push to origin:
  bookmark: push-plrtzxvnuusk [add to c2395e72]
git: Enumerating objects: 17, done.
git: Counting objects: 100% (17/17), done.
git: Delta compression using up to 32 threads
git: Compressing objects: 100% (14/14), done.
git: Writing objects: 100% (16/16), 25.21 KiB | 6.30 MiB/s, done.
git: Total 16 (delta 1), reused 5 (delta 0), pack-reused 0 (from 0)
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
remote:
remote: Create a pull request for 'push-plrtzxvnuusk' on GitHub by visiting:
remote:      https://github.com/mayalin/sync-service/pull/new/push-plrtzxvnuusk
remote:
```

Inspect `p` in the log:

```text
$ jj log -r p
○  plrtzxvn maya@example.com 2026-11-15 13:33:49 push-plrtzxvnuusk c2395e72
│  Run sync worker in pipeline
~
```

> [!TIP]
> You can customize the generated bookmark prefix in your user config with `templates.git_push_bookmark`, for example formatting auto-bookmarks as `"maya/" ++ change_id.short()`. Because `jj git push -c` ties a bookmark to a change ID, it naturally encourages a clean **one logical change per pull request** discipline.

### Updating a remote branch: two workflows

When review feedback arrives on a pull request, you have two ways to update the bookmark.

#### 1. Bookmark squash flow (one evolving change per PR)

Because a bookmark tracks a **change ID**, squashing new edits into a bookmarked revision automatically updates the bookmark in place:

```text
$ jj new push-plrtzxvnuusk
Working copy  (@) now at: rtzvpxow d25b1da8 (empty) (no description set)
Parent commit (@-)      : plrtzxvn c2395e72 push-plrtzxvnuusk | Run sync worker in pipeline
Added 2 files, modified 0 files, removed 0 files

# Edit README.md, then check status:
$ jj status
Working copy changes:
M README.md
Working copy  (@) : rtzvpxow 7cee54ed (no description set)
Parent commit (@-): plrtzxvn c2395e72 push-plrtzxvnuusk | Run sync worker in pipeline

# Squash the edits into the bookmarked parent:
$ jj squash --from @ --into push-plrtzxvnuusk
Working copy  (@) now at: ryuuprxx 15e76b14 (empty) (no description set)
Parent commit (@-)      : plrtzxvn bfc970bf push-plrtzxvnuusk* | Run sync worker in pipeline
```

Notice the asterisk on `push-plrtzxvnuusk*`: it indicates the local bookmark has diverged from `push-plrtzxvnuusk@origin`. Push the updated bookmark:

```text
$ jj git push -b push-plrtzxvnuusk
Changes to push to origin:
  bookmark: push-plrtzxvnuusk [move sideways from c2395e72d528 to bfc970bfa00c]
```

Moving a bookmark "sideways" performs a safe lease-checked force push (`--force-with-lease`). You can preview any push beforehand with `jj git push --dry-run`. Additionally, `jj` protects shared history via its `immutable_heads()` configuration, refusing to rewrite commits on `main` or untracked remote branches unless explicitly overridden.

#### 2. Bookmark advance flow (appending commits to a PR)

If your team prefers appending new commits to a pull request branch rather than amending:

```text
$ jj new push-plrtzxvnuusk
Working copy  (@) now at: ptrztmxp 7f50aa2e (empty) (no description set)
Parent commit (@-)      : plrtzxvn bfc970bf push-plrtzxvnuusk | Run sync worker in pipeline

# Edit README.md and commit:
$ jj commit -m "Add pipeline usage example"
Working copy  (@) now at: unttkrpw 155e3e09 (empty) (no description set)
Parent commit (@-)      : ptrztmxp 3452ff5b Add pipeline usage example
```

Check `jj log -r p::`:

```text
$ jj log -r p::
@  unttkrpw maya@example.com 2026-11-15 13:51:57 155e3e09
│  (empty) (no description set)
○  ptrztmxp maya@example.com 2026-11-15 13:51:57 3452ff5b
│  Add pipeline usage example
○  plrtzxvn maya@example.com 2026-11-15 13:43:16 push-plrtzxvnuusk bfc970bf
│  Run sync worker in pipeline
~
```

Because bookmarks do not move automatically on `jj commit`, `push-plrtzxvnuusk` still points at `plrtzxvn`. Advance the bookmark onto `@-` (`ptrztmxp`) with `jj bookmark advance`:

```text
$ jj bookmark advance --to @-
Advanced 1 bookmarks to ptrztmxp 3452ff5b push-plrtzxvnuusk* | Add pipeline usage example

$ jj git push -b push-plrtzxvnuusk
Changes to push to origin:
  bookmark: push-plrtzxvnuusk [move forward from bfc970bfa00c to 3452ff5bb665]
```

### Fetching remote branches and syncing with trunk

To work on top of a colleague's remote branch, fetch from the remote and create a new working-copy commit on `<bookmark>@origin`:

```bash
jj git fetch
jj new batch-retry-logic@origin
```

> [!NOTE]
> If you run `git status` in a colocated repository while working in `jj`, Git will report `HEAD detached at ...`. That is completely normal: `jj` always checks out the physical commit of `@` rather than activating a named Git branch ref.

Syncing your local work with upstream `main` follows the same pattern—fetch remote changes and rebase your bookmark or stack onto `main`:

```bash
jj git fetch
jj rebase -b my-feature-bookmark -o main
```

## Staying oriented in the graph

Manipulating a DAG is easiest when you can see the graph update live. Three lightweight patterns work well on Linux:

1. **Continuous log monitor (`viddy` or `watch`):** Keep a narrow terminal pane running a live `jj log` with `--ignore-working-copy` (so the monitor itself never triggers concurrent working-copy snapshots):

   ```bash
   viddy -t -n1 jj log --ignore-working-copy --color=always
   ```

2. **Focused work revset:** Configure `revsets.log` in your user TOML config to show `trunk()`, tracked remote bookmarks ahead of trunk, and all mutable commits you own:

   ```text
   visible() & (trunk() | (tracked_remote_bookmarks() & ~::trunk()) | (mutable() & mine())::)
   ```

3. **Interactive TUIs:** Use [Majjit](08-majjit-jj-dag-tui.md) (a Magit-style chord-driven TUI for the `jj` DAG) or [jjui](https://github.com/idursun/jjui) for visual node selection, folding, and rebasing.

## Stacked pull requests in brief

Large features review faster and merge with fewer regressions when decomposed into a sequential stack of small, self-contained changes:

1. **Stage 1:** Refactor existing interfaces to prepare for the feature
2. **Stage 2:** Implement the core feature logic
3. **Stage 3:** Wire the feature into downstream callers

In Git, maintaining a 3-to-5-PR stack is painful: every review comment on Stage 1 forces a cascading sequence of branch checkouts, interactive rebases, and force-pushes across Stages 2 and 3.

In `jj`, a stack is simply a linear chain of commits where each commit carries its own bookmark:

```text
$ jj log -r kpmyzqmr::txummoow
○  txummoow maya@example.com 2026-11-15 14:10:48 maya/txummoow c6e1ba32
│  Wire sync worker into scheduler
○  wqpzktsn maya@example.com 2026-11-15 14:06:27 maya/wqpzktsn ae63d36a
│  Implement record sync worker
○  kpmyzqmr maya@example.com 2026-11-15 14:01:19 maya/kpmyzqmr 18a70828
│  Refactor job dispatcher interface
~
```

When a reviewer asks for a change on Stage 1 (`kpmyzqmr`), you either `jj edit kpmyzqmr` (or `jj new kpmyzqmr`, make the fix, and `jj squash`). `jj` automatically rebases `wqpzktsn` and `txummoow` on top of the updated base commit and updates their attached bookmarks (`maya/wqpzktsn*` and `maya/txummoow*`). One `jj git push --tracked` (or pushing the stack bookmarks) updates every open PR in the chain at once.

## Quick reference: Git vs Jujutsu

| Task | Git command | Jujutsu (`jj`) equivalent |
|---|---|---|
| Initialize in existing Git repo | *(already initialized)* | `jj git init` |
| Inspect working tree & parent | `git status` | `jj status` (`jj st`) |
| Show current commit diff | `git diff HEAD` | `jj diff` / `jj show` |
| Set commit message | `git commit --amend -m "..."` | `jj desc -m "..."` |
| Finalize change & start next | `git add -A && git commit -m "..."` | `jj commit -m "..."` (or `jj desc` + `jj new`) |
| Edit an older commit in a stack | `git rebase -i <hash>^` | `jj edit <change-id>` (descendants auto-rebase) |
| Fold current edits into parent | `git commit --amend --no-edit` | `jj squash` |
| Fold revision X into revision Y | *(manual interactive rebase)* | `jj squash --from X --into Y` |
| Move a commit before another | `git rebase -i` (reorder lines) | `jj rebase -r X --before Y` |
| Drop a commit from a stack | `git rebase -i` (`drop` line) | `jj abandon <change-id>` |
| Split a commit in two | `git rebase -i` (`edit`, `reset HEAD^`, `add -p`) | `jj split -r <change-id>` |
| Undo last repository operation | `git reflog` + `git reset --hard` | `jj undo` (or `jj op log` / `jj op restore`) |
| Push change as a new PR branch | `git checkout -b feat && git push -u origin feat` | `jj git push -c <change-id>` |
| Advance branch pointer to `@-` | `git branch -f feat HEAD` | `jj bookmark advance --to @-` |

## Sources and further reading

- [Official Jujutsu documentation](https://docs.jj-vcs.dev/latest/) and the [Revsets specification](https://docs.jj-vcs.dev/latest/revsets/)
- [Working with GitHub and bookmarks in `jj`](https://docs.jj-vcs.dev/latest/github/)
- [Git compatibility and colocation guide](https://docs.jj-vcs.dev/latest/git-compatibility/)
- [Steve Klabnik's Jujutsu Tutorial](https://steveklabnik.github.io/jujutsu-tutorial/)
- [Jujutsu releases and changelog](https://github.com/jj-vcs/jj/releases/)
