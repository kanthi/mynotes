---
title: 'GoReleaser and GitHub Actions'
---

# GoReleaser and GitHub Actions

Automate reproducible, multi-platform Go releases on every Git tag (`v*.*.*`) using **GoReleaser** and **GitHub Actions**—producing static binaries, SHA-256 checksums, SBOMs, changelogs, and container images in a single pipeline.

## GitHub Actions Release Workflow (`.github/workflows/release.yml`)

```yaml
name: Release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write
  packages: write
  id-token: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version-file: go.mod

      - name: Run tests
        run: go test -race ./...

      - name: Run GoReleaser
        uses: goreleaser/goreleaser-action@v6
        with:
          version: "~> v2"
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Minimal `.goreleaser.yaml`

```yaml
version: 2

before:
  hooks:
    - go mod tidy

builds:
  - main: ./cmd/app
    binary: app
    env:
      - CGO_ENABLED=0
    goos: [linux, darwin, windows]
    goarch: [amd64, arm64]
    flags: [-trimpath]
    ldflags:
      - -s -w
      - -X main.version={{.Version}}
      - -X main.commit={{.ShortCommit}}
      - -X main.date={{.Date}}

archives:
  - format: tar.gz
    format_overrides:
      - goos: windows
        format: zip

checksum:
  name_template: "checksums.txt"

sboms:
  - artifacts: archive

changelog:
  sort: asc
  filters:
    exclude: ["^docs:", "^test:", "^chore:"]
```

## Local Dry-Run Before Tagging

Always validate your release config locally before pushing a tag:

```bash
goreleaser check
goreleaser release --snapshot --clean
```

Inspect `dist/` to verify binary sizes, `checksums.txt`, and `app --version` output.

## Related Chapters

- [335. Distributing Your CLI](../09-cli-tools-in-go/335-distributing-go-clis.md) — Homebrew, Chocolatey, Docker, and `.deb`/`.rpm` packaging.
- [70. Docker Multi-Stage Builds](70-docker-multistage.md)
