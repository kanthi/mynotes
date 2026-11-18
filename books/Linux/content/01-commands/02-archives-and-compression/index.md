---
title: Intro
---

# Intro

Pack, compress, and inspect archives for backup, distribution, and log history. Prefer `tar` with modern compressors (`zstd`, `pigz`, `xz`) for Linux trees; use `zip` or `7z` when exchanging with non-Unix tools.

## Commands in this part

| Command | Role |
|---------|------|
| `tar` | tar packs multiple files into one archive (and extracts them). |
| `gzip` | gzip compresses or decompresses files using the DEFLATE algorithm (.gz). |
| `xz` | xz compresses and decompresses files in the XZ format (LZMA2), typically yielding better ratios than gzip/bzip2 at… |
| `bzip2` | bzip2 compresses files with the Burrows–Wheeler algorithm (.bz2). |
| `zstd` | zstd (Zstandard) is a modern compression tool offering an excellent speed vs ratio curve, levels from very fast to… |
| `zip` | zip creates ZIP archives widely used for cross-platform interchange (Windows, macOS, Linux). |
| `unzip` | unzip lists, tests, and extracts ZIP archives. |
| `zcat` | zcat writes gzip-compressed files to stdout as if they were uncompressed (equivalent to gzip -dc). |
| `zgrep` | zgrep runs grep over gzip-compressed files without manual decompression. |
| `7z` | 7z creates and extracts high-ratio `.7z`, `.zip`, and `.iso` archives with optional AES-256 header encryption. |
| `pigz` | pigz is a multi-core parallel implementation of gzip for fast `.gz` backups. |

## Which tool should I use?

| Need | Best Choice | Why |
|------|-------------|-----|
| Fast Linux backups (best speed/ratio) | `tar -I zstd -cf out.tar.zst dir/` | Multi-threaded, preserves Unix permissions |
| Multi-core `.gz` compatibility | `tar -I pigz -cf out.tar.gz dir/` | Readable by every `gzip`/`gunzip` |
| Maximum ratio for static distribution | `tar -cJf out.tar.xz dir/` | Smallest tarball; slower to compress |
| Windows/macOS sharing | `zip` / `unzip` (or `7z`) | Built into desktop OSes |
| Search/read rotated `.gz` logs in place | `zgrep` / `zcat` | No temp files on disk |

## Suggested starting points

1. Create/extract trees: `tar` (+ `zstd`/`pigz`/`gzip`/`xz`).
2. Single-file compressors: `zstd`, `pigz`, `gzip`, `xz`, `bzip2`.
3. Interchange with Windows/macOS users: `zip` / `unzip` / `7z`.
4. Search rotated logs without unpacking: `zgrep`, `zcat`.

## Related parts

- Files and paths — copy and verify before archiving
- Text and pipes — process extracted streams
- Networking — `rsync`/`scp` for transfer after packing

Continue with the individual command pages in this part.
