---
title: Shallow, Partial, and Sparse Clones
---

# Shallow, Partial, and Sparse Clones

A standard `git clone` downloads every commit, every tree, and every version of every file ever committed, then checks out the entire directory tree onto disk. On a 20-year-old monorepo with millions of objects, that default takes tens of minutes and dozens of gigabytes. Modern Git gives you three orthogonal levers to download and materialize only what you need.

## Comparing the Three Scale Levers

| Technique | Command Flag | What Is Omitted | Best Use Case |
| :--- | :--- | :--- | :--- |
| **Shallow clone** | `--depth=1` | Commit history older than N commits | Ephemeral CI jobs that only build `HEAD` |
| **Blobless partial clone** | `--filter=blob:none` | Historical file blobs (fetched on demand) | Daily developer clones of huge repositories |
| **Treeless partial clone** | `--filter=tree:0` | Historical trees and blobs (fetched on demand) | Fast throwaway builds needing full commit graph |
| **Sparse checkout** | `git sparse-checkout set <dir>` | Unneeded working-tree directories on disk | Working on one service inside a monorepo |

## Combining Blobless Partial Clone with Cone-Mode Sparse Checkout

The fastest developer workflow for a massive monorepo combines `--filter=blob:none` (keeps full commit history for `git log` and `git blame`, downloads blobs lazily) with **cone-mode sparse checkout** (only materializes the directories you actively edit, reducing `git status` scan time from seconds to milliseconds):

```bash
# 1. Clone commit/tree metadata without historical blobs or checking out all files
git clone --filter=blob:none --sparse <repo-url> monorepo
cd monorepo

# 2. Materialize only the specific service and shared libraries you need
git sparse-checkout set --cone services/billing libs/auth

# 3. Verify active sparse patterns
git sparse-checkout list
```

### Enabling the Sparse Index

In repositories with hundreds of thousands of paths, pair cone mode with the **sparse index** so `.git/index` collapses outside directories into single tree entries instead of listing every skipped file:

```bash
git config index.sparse true
git config core.untrackedCache true
```

> [!WARNING]
> Avoid `--depth=1` shallow clones for interactive developer work or pushes. Shallow repositories break `git merge-base` calculations across diverged branches and force expensive object renegotiation on `git fetch`. Prefer `--filter=blob:none` whenever you need history or branching.

## Hands-On Lab: Measure Disk Savings with Sparse Checkout

Create a local multi-service repository and verify how cone-mode sparse checkout trims the working tree without losing Git history:

```bash
mkdir -p /tmp/vcs-scale-lab/origin && cd /tmp/vcs-scale-lab/origin
git init -b main
mkdir -p services/api services/web services/ml
for d in services/api services/web services/ml; do
  echo "module $d" > "$d/README.md"
done
git add -A && git commit -m "initial monorepo layout"

# Clone with sparse checkout enabled
cd /tmp/vcs-scale-lab
git clone --sparse /tmp/vcs-scale-lab/origin dev-checkout
cd dev-checkout
ls -la services/ 2>/dev/null || echo "Only root files checked out initially"

# Materialize only services/api
git sparse-checkout set --cone services/api
find . -maxdepth 3 -not -path './.git*'
```
