---
title: Install the CLI, create a project, and run your first preview
---

# Install the CLI, create a project, and run your first preview

Getting productive with Pulumi starts with mastering the local loop: installing the CLI, choosing where stack state lives, scaffolding a reproducible Python virtual environment, and running `preview`, `up`, `refresh`, and `destroy` with confidence. This chapter walks through the mechanics behind each command and ends with a complete local lab that exercises drift, diffing, and teardown without touching a paid cloud account.

## The infrastructure lifecycle at a glance

Every Pulumi workflow revolves around a single durable artifact: the stack's **state checkpoint**. Commands either read that checkpoint, update it after mutating live infrastructure, or reconcile it against real-world drift.

![Pulumi infrastructure lifecycle showing pulumi new, pulumi preview, pulumi up, pulumi refresh, and pulumi destroy interacting with the stack state checkpoint.](../../images/diagram-pulumi-lifecycle.svg)

| Command | Mutates cloud? | Mutates state? | Purpose |
| :--- | :---: | :---: | :--- |
| `pulumi new` | No | Initializes empty stack | Scaffolds `Pulumi.yaml`, runtime config, and entrypoint |
| `pulumi preview` | No | No | Dry-run diff between program and last checkpoint |
| `pulumi up` | Yes | Yes | Converges live resources to match the program |
| `pulumi refresh` | No | Yes | Reads live provider state to capture out-of-band drift |
| `pulumi destroy` | Yes | Yes | Deletes all managed resources in reverse DAG order |

## Installing the Pulumi CLI

The `pulumi` binary bundles the CLI, the deployment engine, and plugin management. Language SDKs (`pulumi`, `pulumi-aws`, `pulumi-random`) are installed separately inside your project's package manager (`pip`, `uv`, `npm`, `go mod`).

### Linux and macOS (official installer)

```bash
curl -fsSL https://get.pulumi.com | sh
export PATH="$HOME/.pulumi/bin:$PATH"
```

Add the `export PATH=...` line to `~/.bashrc` or `~/.zshrc` so new shells find the binary automatically.

### Homebrew (macOS and Linux)

```bash
brew install pulumi/tap/pulumi
```

### Verify the binary and installed plugins

```bash
pulumi version
pulumi plugin ls
```

`pulumi plugin ls` lists cached resource and language plugins under `~/.pulumi/plugins`. Whenever your program imports a provider package (such as `pulumi-random`), the CLI downloads the matching provider binary automatically during the first `preview` or `up`.

## Choosing and authenticating a state backend

Before you can create a stack, Pulumi needs a **backend** to store checkpoints and stack metadata. You select a backend with `pulumi login`.

| Backend | Command | Where state lives | Locking & history | Typical use case |
| :--- | :--- | :--- | :--- | :--- |
| **Pulumi Cloud** | `pulumi login` | Managed Pulumi Cloud (`app.pulumi.com`) | Transactional locking, audit logs, RBAC | Production teams and CI/CD |
| **Local filesystem** | `pulumi login --local` (or `file://~`) | `~/.pulumi/stacks` | Local file lock only | Learning, offline labs, single-dev spikes |
| **AWS S3 / GCS / Azure Blob** | `pulumi login s3://my-state-bucket` | Your cloud object store | Object-lock / lease file | Strict self-hosted compliance |
| **PostgreSQL** | `pulumi login postgres://...` | Self-hosted Postgres | Database transactions | Air-gapped or on-prem teams |

Check which backend and identity are currently active at any time:

```bash
pulumi whoami -v
```

> [!TIP]
> **Passphrase prompt on DIY backends**
>
> Self-managed backends (`--local`, `s3://`, `gs://`) encrypt stack secrets locally using a passphrase by default. For local labs, set `export PULUMI_CONFIG_PASSPHRASE="lab-passphrase"` in your shell so `preview` and `up` do not prompt interactively on every run. Never reuse a weak passphrase for production stacks — use KMS or Vault instead (covered in [Configuration and Secrets](03-configuration-and-secrets.md)).

## Anatomy of a Python Pulumi project

Scaffolding a Python project with `pulumi new` creates a self-contained directory:

```text
demo-random/
├── Pulumi.yaml        # Project name, runtime, toolchain, optional backend URL
├── Pulumi.dev.yaml    # Stack-specific config & encrypted secrets for `dev`
├── requirements.txt   # Python SDK and provider packages
├── venv/              # Isolated Python virtual environment
└── __main__.py        # Entrypoint executed by the Python language host
```

### `Pulumi.yaml`

```yaml
name: demo-random
runtime:
  name: python
  options:
    toolchain: pip
    virtualenv: venv
description: Minimal Python Pulumi project using the random provider
```

Setting `options.virtualenv: venv` tells the Pulumi language host to invoke `./venv/bin/python` automatically, even if you forgot to run `source venv/bin/activate` in your terminal. You can also set `toolchain: uv` or `toolchain: poetry` if your team prefers those managers.

### `requirements.txt`

```text
pulumi>=3.100.0,<4.0.0
pulumi-random>=4.16.0,<5.0.0
```

### `__main__.py`

```python
import pulumi
import pulumi_random as random

# Logical name "app-token" uniquely identifies this resource in state
token = random.RandomPassword(
    "app-token",
    length=24,
    special=True,
    override_special="_@#%",
)

# Physical names get auto-suffixed unless you override them;
# RandomPet generates readable identifiers for dev/staging environments
label = random.RandomPet(
    "env-label",
    length=2,
    separator="-",
)

pulumi.export("environment_label", label.id)
pulumi.export("app_token", token.result)
```

Notice the distinction between the **logical name** (`"app-token"`) and the **physical resource**. Pulumi tracks resources in state by their URN, which includes the logical name. Changing `"app-token"` to `"service-token"` tells Pulumi to create a brand-new resource and delete the old one unless you attach an `Alias`.

## Reading `pulumi preview` like an operator

Always run `pulumi preview --diff` before applying changes to any shared environment. The diff symbols tell you the exact blast radius:

| Symbol | Operation | Meaning |
| :--- | :--- | :--- |
| `+` | `create` | Resource exists in code, not in state — will be created |
| `~` | `update` | Mutable property changed — updated in place with no downtime |
| `+-` | `replace` | Immutable property changed — creates replacement and deletes old resource |
| `-` | `delete` | Resource removed from code — will be destroyed |
| `=` | `same` | No change detected against state |

> [!WARNING]
> **Watch for `+-` replacements on stateful resources**
>
> Changing a database engine version, subnet CIDR, or logical resource name can trigger a replacement (`+-`). By default Pulumi creates the new resource before deleting the old one (`create-before-replace`), except when a fixed physical name would collide — in which case it deletes first. Always inspect replacements carefully in `pulumi preview --diff`.

## Multi-stack workflow and safe teardown

One project can drive as many isolated stacks as you need:

```bash
pulumi stack init staging          # Create and select a new stack
pulumi stack init prod             # Create and select prod
pulumi stack ls                    # '*' marks the currently active stack
pulumi stack select dev            # Switch back to dev
pulumi stack output --json         # Machine-readable exported outputs
```

When an environment is no longer needed, destroy its live resources first and only then remove the empty stack metadata:

```bash
pulumi destroy --yes               # Delete all managed resources in reverse order
pulumi stack rm dev --yes          # Remove empty stack history and Pulumi.dev.yaml
```

> [!CAUTION]
> Never pass `--force` to `pulumi stack rm` on a stack that still manages cloud resources. `--force` deletes the state record while leaving the live cloud infrastructure running and orphaned.

## Common pitfalls for beginners

### 1. Concatenating `Output[str]` with plain strings

Resource attributes are asynchronous `Output` values, not plain Python strings:

```python
# WRONG: embeds an Output object warning instead of the actual secret value
bad_url = "https://user:" + token.result + "@db.internal"

# RIGHT: use Output.format() or .apply()
good_url = pulumi.Output.format("https://user:{0}@db.internal", token.result)
```

### 2. Missing virtual environment packages

If `pulumi preview` raises `ModuleNotFoundError: No module named 'pulumi_random'`, verify that `Pulumi.yaml` points `runtime.options.virtualenv` to the exact venv where you ran `pip install`.

### 3. Editing state files by hand

Never hand-edit JSON files under `~/.pulumi/stacks/` or S3. Use `pulumi state delete`, `pulumi state unprotect`, or `pulumi refresh` so Pulumi maintains referential integrity across dependencies.

## Hands-on lab: complete local lifecycle (preview, up, update, replace, destroy)

This lab walks you through every state transition (`+ create`, `~ update`, `+- replace`, and `- delete`) locally in under five minutes.

### Step 1: Initialize the local workspace

```bash
mkdir -p ~/pulumi-labs/02-getting-started && cd ~/pulumi-labs/02-getting-started
export PULUMI_CONFIG_PASSPHRASE="local-lab-passphrase"
pulumi login --local

pulumi new python \
  --name lifecycle-lab \
  --description "End-to-end Pulumi lifecycle lab" \
  --stack dev \
  --yes

./venv/bin/pip install pulumi-random
```

### Step 2: Create initial resources (`+ create`)

Write `__main__.py`:

```python
import pulumi
import pulumi_random as random

cluster_pet = random.RandomPet("cluster", length=2, separator="-")
db_pass = random.RandomPassword(
    "db-password",
    length=20,
    special=True,
    override_special="!@#%",
)

pulumi.export("cluster_name", cluster_pet.id)
pulumi.export("db_password", db_pass.result)
```

Preview and apply:

```bash
pulumi preview --diff
pulumi up --yes
pulumi stack output
pulumi stack output db_password --show-secrets
```

Notice that `pulumi stack output` masks `db_password` as `[secret]`, while `--show-secrets` decrypts it on demand.

### Step 3: Trigger a replacement (`+- replace`)

Edit `__main__.py` to change `length=20` to `length=32` on `db_pass`, then run:

```bash
pulumi preview --diff
```

Because a password's `length` is immutable, Pulumi plans `+- replace` for `db-password` while leaving `cluster` untouched (`= same`). Apply the change:

```bash
pulumi up --yes
```

### Step 4: Inspect a second isolated stack (`staging`)

```bash
pulumi stack init staging
pulumi up --yes
pulumi stack output cluster_name -s dev
pulumi stack output cluster_name -s staging
```

Both stacks run the exact same `__main__.py`, yet each generates and remembers its own independent `cluster_name` and `db_password`.

### Step 5: Tear down both stacks cleanly

```bash
pulumi destroy -s staging --yes && pulumi stack rm staging --yes
pulumi destroy -s dev --yes && pulumi stack rm dev --yes
```

## Summary

- The Pulumi CLI drives the deployment engine, downloads provider plugins, and reads/writes stack checkpoints in your configured backend.
- `Pulumi.yaml` pins your project name, language runtime, and virtual environment so runs stay reproducible across machines.
- `pulumi preview --diff` reveals creates, in-place updates, replacements, and deletes before anything touches live infrastructure.
- Always run `pulumi destroy` before `pulumi stack rm` so you never orphan live resources.

## Sources

- [Pulumi Installation Guide](https://www.pulumi.com/docs/install/)
- [Pulumi Projects](https://www.pulumi.com/docs/iac/concepts/projects/)
- [CLI Reference: `pulumi new`](https://www.pulumi.com/docs/iac/cli/commands/pulumi_new/)
- [CLI Reference: `pulumi preview`](https://www.pulumi.com/docs/iac/cli/commands/pulumi_preview/)
- [CLI Reference: `pulumi up`](https://www.pulumi.com/docs/iac/cli/commands/pulumi_up/)
