---
title: '335. Distributing Your CLI'
---

# Distributing Your CLI

A Go binary is easy to compile locally, but real adoption depends on frictionless installation across macOS, Linux, Windows, and containers. **GoReleaser** paired with **GitHub Actions** automates cross-compilation, checksums, Docker images, Homebrew taps, Chocolatey packages, and `.deb`/`.rpm` archives from a single Git tag.

## Module Syllabus (10 Lessons)

1. Introduction
2. Release Automation
3. Containerizing with Docker
4. Automated Cross-Compilation (GoReleaser + GitHub Actions)
5. Automated Homebrew distribution
6. Automated Chocolatey distribution
7. Automated Docker distribution
8. Automated RPM and Deb files
9. Exercise
10. Exercise Solution

## Release Pipeline Architecture

```text
git tag v1.2.0 && git push --tags
        |
        v
GitHub Actions (.github/workflows/release.yml)
        |
        v
   GoReleaser (.goreleaser.yaml)
     ├── Cross-compile (linux/darwin/windows × amd64/arm64, CGO_ENABLED=0)
     ├── Generate SHA256 checksums + GitHub Release notes
     ├── Publish Homebrew formula (macOS / Linuxbrew)
     ├── Publish Chocolatey package (Windows)
     ├── Build & push multi-arch Docker image (GHCR / Docker Hub)
     └── Package .deb and .rpm via nFPM
```

## Multi-Platform `.goreleaser.yaml` Blueprint

```yaml
version: 2

builds:
  - main: ./cmd/urlmon
    binary: urlmon
    env:
      - CGO_ENABLED=0
    goos: [linux, darwin, windows]
    goarch: [amd64, arm64]
    ldflags:
      - -s -w
      - -X example.com/urlmon/cmd.Version={{.Version}}
      - -X example.com/urlmon/cmd.Commit={{.ShortCommit}}
      - -X example.com/urlmon/cmd.Date={{.Date}}

archives:
  - format: tar.gz
    format_overrides:
      - goos: windows
        format: zip

nfpms:
  - id: packages
    package_name: urlmon
    homepage: https://example.com/urlmon
    maintainer: Maintainer <dev@example.com>
    description: Check and monitor HTTP endpoints from the terminal
    license: MIT
    formats: [deb, rpm]

brews:
  - name: urlmon
    repository:
      owner: your-org
      name: homebrew-tap
    homepage: https://example.com/urlmon
    description: Check and monitor HTTP endpoints from the terminal

chocolateys:
  - name: urlmon
    authors: your-org
    project_url: https://example.com/urlmon
    summary: Check and monitor HTTP endpoints from the terminal

dockers:
  - image_templates:
      - "ghcr.io/your-org/urlmon:{{ .Version }}"
      - "ghcr.io/your-org/urlmon:latest"
    dockerfile: Dockerfile
```

## Minimal Multi-Stage `Dockerfile`

```dockerfile
FROM golang:1.27-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/urlmon ./cmd/urlmon

FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /out/urlmon /urlmon
ENTRYPOINT ["/urlmon"]
```

## Related Chapters

- Previous: [334. Error Handling & Logging in CLIs](334-cli-error-handling-and-log-rotation.md)
- Foundations: [315. Shipping, Version & Completion](315-shipping-version-completion.md), [76. GoReleaser & GitHub Actions](../20-infrastructure/76-goreleaser-github-actions.md), [326. CLI Packaging & Self-Updates](326-cli-packaging-self-update-mldsa.md)
