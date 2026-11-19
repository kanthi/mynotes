---
name: octavo-books
description: Rules for writing and changing the books in this mynotes repo, which is an Octavo library. Use before adding, editing, moving, renaming or deleting any chapter, part, image or book under books/, before touching an octavo.yml, octavo-library.yml, octavo-pages.txt or the publish workflow, and before running octavo render, preview, check or publish.
---

# Writing in the mynotes Octavo library

This repo is an [Octavo](https://github.com/kanthi/octavo) **library**. Every folder in `books/` that has an `octavo.yml` is a book. Octavo builds each book into a website, an EPUB and a PDF, and builds a portal page that links to all of them. The full user guide is available through `octavo guide`. Its source is in `guide/content/` of the Octavo repo, if a local copy exists (for example `~/Workspace/Repos/octavo`).

## Layout

```text
octavo-library.yml          library settings (title, books-dir: books, output-dir: public)
books/<Book>/
  octavo.yml                book settings and the chapter list
  octavo-pages.txt          every page address the site has had (managed by Octavo, committed)
  index.md                  the preface: the book's first page, unnumbered
  content/NN-part/          one numbered folder per part (or section)
    index.md                the part's intro page (front matter `title: Intro`)
    NN-topic.md             one chapter per file
  images/                   diagrams, shared by the whole book
  _book/                    build output, never edited, never committed
public/                     library build output, never edited, never committed
.github/workflows/octavo.yml  builds only the changed books and publishes them to GitHub Pages on push to main
```

## Rules for chapters

1. **One chapter per Markdown file.** The first line is `# Title`, a level-1 heading, and it becomes the chapter title in the sidebar, the browser tab, the EPUB and the PDF. Inside a chapter, use only `##` and `###`. Those two levels feed the "On this page" list.
2. **Give every new file a number prefix** that places it in its folder, such as `21-shuf.md` or `05a-extra.md`. A numbered file joins the book by itself (`auto-add`) and goes right after the file in the same folder with the next lower number. A numbered new folder becomes a new part. **You do not need to edit `octavo.yml` for a numbered file.** Edit it only to give a chapter a fixed place or a sidebar label (`text:` plus `file:`).
3. **A file without a number is left out** and causes a warning. To keep a file in the repo but out of the book, start its name with `_`, as in `_notes.md` or `_planning/`. `README.md` is always ignored.
4. **To remove a chapter**, delete it or rename it with a `_` prefix. Taking it out of `octavo.yml` is not enough, because its number brings it back.
5. **Never reuse a number in a way that reorders existing pages unless asked.** Append new pages with the next free number.
6. **Front matter** (optional) supports only `title:` (it overrides the `#` heading) and `unnumbered: true` (for appendices and glossaries).
7. **Drafts are whole books**: `draft: true` under `book:`. Builds leave a draft out unless `--drafts` is passed. Chapters cannot be drafts.

## Rules for Markdown

- CommonMark with GitHub additions: tables, task lists, strikethrough, footnotes (`[^id]`) and autolinks.
- Always give a code block a language (` ```bash `, ` ```go `, ` ```yaml `, and so on). Use ` ```text ` for output and file trees.
- **Callouts** use GitHub syntax, with the marker alone and in capitals on the first line:
  `> [!NOTE]`, `> [!TIP]`, `> [!IMPORTANT]`, `> [!WARNING]`, `> [!CAUTION]`.
- **Link between chapters with the `.md` file path**, relative to the current file: `[sed](02-sed.md)` or `[pipes](../13-text-and-pipes/index.md#pipeline-recipes)`. Octavo rewrites it for every format. Never link to `.html` files or `_book/`. Anchors are the heading text in lower case, with spaces as hyphens.
- **Images** are relative to the chapter, so a page two folders deep uses `../../images/x.svg`. Always write meaningful alt text.
- **Dark-mode diagrams:** put `name-dark.svg` next to `name.svg` and reference **only** `name.svg`. Octavo switches to the dark variant on dark pages. Name diagrams `diagram-<topic>.svg`. SVGs carry `role="img"` and a descriptive `aria-label`.
- Raw HTML (such as `<details>`) works, but use it rarely.

## House style for pages

Match the pages already in the same folder. Read one or two sibling files before writing. The usual shapes:

- **Command or tool page** (for example `books/Linux/content/01-commands/**`): `# name`, then `## Overview`, `## Syntax`, `## Common Options` (a table), `## Key Use Cases`, `## Examples with Explanations` (with `###` per example: a code block, then one or two sentences of explanation), `## Notes & Pitfalls`, `## Related Commands`, `## Additional Resources`.
- **Part `index.md`**: front matter `title: Intro`, then `# Intro`, a short summary, a table of the part's pages with their role, and guidance such as "which tool should I use" or recipes.
- **When you add a page to a part, also update that part's `index.md`.** Add the page to the table and to any guide or recipe lists, so the intro stays complete.
- Examples should be correct and runnable. Explain *why*, not only *what*. Point out portability traps, such as GNU vs BSD/macOS.

## Rules for config and generated files

- `octavo-pages.txt`: **never edit it by hand, and always commit it** when Octavo changes it. It keeps old addresses working after a move or rename. If you rename **and** retitle a chapter, add `format.html.redirects` in that book's `octavo.yml` (`old/path.html: new/path.md`).
- `_book/` and `public/` are build output, and they are in `.gitignore`. Never edit them, and never commit them.
- Add new books with `octavo add <Name>` (or `--from <path>` to copy a book in). Never use `octavo new` inside the library. A book name may use letters, digits, `-`, `_` and `.`. After adding a book, add a row to the table in the root `README.md`.
- Publish targets belong in `octavo-library.yml`, not in a book's `octavo.yml`.
- Do not change `.github/workflows/octavo.yml` unless asked. It detects changed books from `books/<Name>/` paths and does a full rebuild for any change outside `books/`.

## Commands

Run these from the repo root. If `octavo` is not on `PATH`, use a local build (for example `~/Workspace/Repos/octavo/octavo`) or install it with `GOPROXY=direct go install github.com/kanthi/octavo/cmd/octavo@latest`.

```sh
octavo check                 # broken links, missing images, unlisted files; builds nothing
octavo render --to html Go   # quick website build of one book (names match in any case)
octavo render                # every book, every format (PDF needs Typst or Chrome)
octavo preview --open        # live preview
```

**Before you finish, run `octavo check`.** It must report **0 errors**. Make sure your change adds no new warnings for the files you touched. Some books already have warnings, and you do not need to fix those unless asked.

## Commits

One logical change per commit. The subject starts with the book name, as in `Linux: add shuf, tac and rev to Text and Pipes`. The body is a short list of what each page covers, plus a line on the section index update. Commit only when the user asks.
