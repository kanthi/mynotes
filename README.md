# My Notes

My notes and training material, written as books with [Octavo](https://github.com/kanthi/octavo). Each book becomes a website, an EPUB and a PDF, and a portal page links them all.

## Books

| Book | Folder |
|---|---|
| Boring Linux | `books/Boring-Linux` |
| BYOL | `books/BYOL` |
| C | `books/C` |
| DevOps with Pulumi | `books/DevOps-Pulumi` |
| Go | `books/Go` |
| Linux | `books/Linux` |
| Maths | `books/Maths` |
| Networking | `books/Networking` |
| NixOS | `books/NixOS` |
| OpenTofu | `books/OpenTofu` |
| Python | `books/Python` |
| Rust | `books/Rust` |
| VCS | `books/VCS` |
| Writing Kernel with AI | `books/Writing-Kernel-with-AI` |

## Build

```sh
octavo render              # every book: website, EPUB and PDF (PDF needs Chrome)
octavo render --to html    # websites only, a few seconds
octavo render Go Rust      # only these books
octavo preview --open      # live preview while writing
octavo check               # broken links, missing images, files left out
```

The result is in `public/`: the portal at `public/index.html`, each website in `public/html/<Book>/`, and the files in `public/pdf/` and `public/epub/`. `public/` and each book's `_book/` folder are build output, not kept in git.
