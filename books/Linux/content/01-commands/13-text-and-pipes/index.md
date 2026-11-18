---
title: Intro
---

# Intro

Filter, reshape, and measure text streams — the core of shell pipelines. Line tools (`grep`/`sed`/`awk`) plus structured JSON/YAML (`jq`, `yq`), unit formatting (`numfmt`), and progress (`pv`).

## Commands in this part

| Command | Role |
|---------|------|
| `grep` | grep searches input lines for a pattern. |
| `sed` | sed (stream editor) transforms text line by line — substitute, delete, print, and simple inserts — without opening… |
| `awk` | awk is a pattern–action language for column-oriented text. |
| `cut` | The cut command extracts specific columns or fields from lines of text. |
| `sort` | sort orders lines of text (or selected fields) and can merge pre-sorted files. |
| `uniq` | uniq reports or filters adjacent duplicate lines. |
| `tr` | The tr (translate) command translates or deletes characters from standard input. |
| `head` | The head command displays the first lines of files or input streams. |
| `tail` | The tail command displays the last lines of files or input streams. |
| `wc` | The wc (word count) command counts lines, words, characters, and bytes in files or input streams. |
| `tee` | tee reads stdin and writes it to both standard output and one or more files. |
| `xargs` | xargs builds and executes command lines from standard input. |
| `diff` | The diff command compares files line by line and displays the differences. |
| `comm` | comm compares two sorted files line by line and outputs three columns: lines only in file1, only in file2, and… |
| `paste` | paste merges lines of files side by side, writing corresponding lines separated by tabs (or another delimiter). |
| `column` | column formats text into aligned columns or a table. |
| `jq` | jq is a command-line JSON processor. |
| `pv` | pv (Pipe Viewer) shows a progress bar, throughput, and ETA for data moving through a pipeline. |
| `yq` | yq is a command-line processor for YAML, TOML, XML, and JSON configuration files. |
| `numfmt` | numfmt converts numbers to and from human-readable units (`Ki`, `Mi`, `Gi`) inside pipelines. |

## Which tool should I use?

| Need | Best Choice | Why |
|------|-------------|-----|
| Filter matching lines | `grep` (or `rg` for trees) | Fast line matching |
| Simple search-and-replace | `sed 's/old/new/g'` | Lightweight stream edits |
| Multi-column math, sums, or reordered fields | `awk` | Native field splitting (`$1`, `$NF`) and variables |
| Fixed delimiter slice | `cut -d, -f1,3` | Faster and simpler than `awk` for strict delimiters |
| Structured JSON | `jq` | Never parse JSON with `grep`/`sed`/`awk` |
| Structured YAML / TOML / XML | `yq` | Preserves YAML formatting and comments with `-i` |
| Human-readable byte columns | `numfmt` | Formats raw bytes (`--to=iec-i`) while preserving headers |
| Unique frequency counts | `sort \| uniq -c \| sort -nr` | `uniq` only collapses *adjacent* duplicates |

## Pipeline Recipes

```bash
# 1. Top 10 client IPs in an Nginx access log
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -n 10

# 2. Extract failing pods/services from JSON output with jq
kubectl get pods -o json | jq -r '.items[] | select(.status.phase != "Running") | .metadata.name'

# 3. Monitor throughput while streaming a compressed DB restore
pv dump.sql.gz | gzip -dc | psql -U postgres appdb

# 4. Format top memory processes in readable MiB/GiB
ps -eo pid,comm,rss --sort=-rss | head -n 11 | numfmt --header --to=iec-i --from-unit=1024 --field=3
```

## Suggested starting points

1. Filter: `grep` (or `rg` in Files for recursive code search).
2. Rewrite: `sed`; fields/reports: `awk`.
3. Columns and sets: `cut`, `sort`, `uniq`, `comm`, `paste`, `column`, `numfmt`.
4. Edges of streams: `head`, `tail`, `tee`, `xargs`.
5. Compare: `diff`; JSON/YAML: `jq`, `yq`; progress: `pv`.

## Related parts

- Files and paths — sources of text
- Shell commands — quoting and `printf`
- Scripting → Bash (this book) — programming beyond one-liners

Continue with the individual command pages in this part.
