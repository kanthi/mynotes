---
unnumbered: true
---

# DevOps with Pulumi

Infrastructure as code stops feeling like a separate, brittle scripting world the moment you can build, test, and package cloud infrastructure with the same language, type checker, and test runner you already use for application code. This book teaches **Pulumi** from first principles through production workflows — starting with the deployment engine's gRPC architecture and ending with embedded Automation API platforms and dynamic OIDC credentials via Pulumi ESC.

> [!TIP]
> **Zero-cloud-cost lab philosophy**
>
> Every chapter includes a hands-on lab you can execute locally using `pulumi login --local`, the `pulumi_random` provider, or Pulumi's in-memory mock engine — no paid cloud account or static IAM keys required.

## Who this book is for

- **DevOps and platform engineers** moving from ad-hoc cloud scripts or HCL-only workflows to general-purpose infrastructure languages.
- **Software engineers** who own their services end-to-end and want unit-tested, reusable infrastructure abstractions instead of copy-pasted YAML.
- **Technical leads** evaluating Pulumi's state model, secrets propagation, Automation API, and ESC for multi-environment teams.

## How the book is organized

| Part | Chapter | What you will master |
| :--- | :--- | :--- |
| **Intro** | [What Pulumi is (and is not)](content/00-intro/01-overview.md) | Language host, gRPC deployment engine, providers, state, and multi-language SDKs |
| **Foundations** | [Install the CLI, create a project, and run your first preview](content/01-foundations/01-getting-started.md) | CLI install, backends, `Pulumi.yaml`, and the `preview` / `up` / `refresh` / `destroy` loop |
| **Foundations** | [Projects, Stacks and State](content/01-foundations/02-projects-and-stacks.md) | Stack isolation, `StackReference` contracts, DIY vs Pulumi Cloud state, and drift |
| **Foundations** | [Configuration and Secrets](content/01-foundations/03-configuration-and-secrets.md) | Namespaced config, typed getters, secret tainting, KMS/Vault/passphrase providers |
| **Foundations** | [Inputs, Outputs and the Dependency Graph](content/01-foundations/04-inputs-and-outputs.md) | The `Output[T]` async model, `.apply()`, `Output.all()`, JSON helpers, and DAG ordering |
| **Foundations** | [Unit Testing Pulumi Programs with Mocks](content/01-foundations/05-unit-testing-with-mocks.md) | In-memory `pulumi.runtime.Mocks`, `pytest`/`unittest` async patterns, security invariants |
| **Foundations** | [Component Resources](content/01-foundations/06-component-resources.md) | `ComponentResource`, URN hierarchy, `parent=self`, `transforms`, `Alias` migrations |
| **Workflows** | [Automation API: drive pulumi up from Python](content/02-workflows/01-automation-api.md) | Embedding `LocalWorkspace` and `Stack` inside CLIs, APIs, and internal developer platforms |
| **Workflows** | [Pulumi ESC: environments, secrets, and dynamic credentials](content/02-workflows/02-pulumi-esc.md) | Hierarchical environment composition, OIDC dynamic logins, `esc run`, and stack bindings |

## Prerequisites

- **Python 3.11+** (primary language for code listings and labs; TypeScript and Go comparisons appear where helpful).
- **Pulumi CLI v3.x** (installed in Chapter 1 of Foundations; unit-testing labs in Chapter 5 run without the CLI).
- Familiarity with basic cloud concepts (networking, IAM, object storage) and a POSIX terminal.

## Formats

This book is published as part of the library portal in **HTML** (with automatic light and dark diagram themes), **PDF**, and **EPUB**.
