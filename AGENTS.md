# Agent instructions

This repo is an Octavo library of books. Before you change anything under `books/` or any Octavo file, read and follow [`.claude/skills/octavo-books/SKILL.md`](.claude/skills/octavo-books/SKILL.md). It covers chapter numbering, links, images, callouts, page style, the files you must never edit, and how to verify a change with `octavo check`.

## Commit rules

- Every commit message must clearly describe the work done in that commit. Generic messages such as "update books", "octavo create", or "add chapters" are not acceptable.
- Use a subject line of the form `<Book>: <what changed>` (for example `NixOS: add deploy-rs rollback chapter to Part 9`).
- Add a body when useful. List the chapters or files added, changed, or removed, and say why.
- Make one commit per book. If a change touches several books, split it into separate commits, each containing only that book's files and described in its own message.
- Keep repo-level changes (`AGENTS.md`, `octavo-library.yml`, `.github`) in their own commit with a matching description.
- Run `octavo check` before committing book changes.
