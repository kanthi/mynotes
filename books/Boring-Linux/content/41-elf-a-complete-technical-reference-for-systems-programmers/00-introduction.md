# ELF: A Complete Technical Reference for Systems Programmers

## The Promise of This Book

Every compiled program, shared library, kernel module, and core dump on modern Linux speaks a single binary language: the **Executable and Linkable Format (ELF)**. Most engineers encounter ELF only through cryptic linker errors (`undefined reference to ...`, `relocation R_X86_64_PC32 can not be used when making a shared object`) or runtime loader failures (`error while loading shared libraries`). This reference removes the mystery completely. By the end of these 26 chapters, you will be able to read raw ELF bytes with a hex dump, trace how `execve()` and `ld-linux.so` construct a live process in virtual memory, write a complete ELF parser in C from scratch, and audit real binaries for hardening gaps.

## How This Book Is Organized

The reference progresses from static on-disk structures to dynamic linking, kernel execution, architecture ABIs, hands-on tooling, and security:

- **Foundations & On-Disk Layout (Chapters 1–8):** Origins, the dual linking/execution view, ELF headers, sections, segments, symbols, relocations, and string tables.
- **Dynamic Linking & Runtime Mechanics (Chapters 9–15):** Dynamic sections, SysV vs. GNU hash tables, PLT/GOT lazy and eager binding, PIC/PIE, Thread-Local Storage (TLS), constructors/destructors, and linker scripts.
- **End-to-End Lifecycle & Kernel Execution (Chapters 16–20):** Source-to-binary build trace, dynamic linker loading sequence, `sys_execve` and the auxiliary vector, notes/build-IDs, and GNU/Linux extensions (RELRO, IFUNC, symbol versioning).
- **Architecture ABIs, Tooling & Implementation (Chapters 21–26):** x86-64 and AArch64 ABI details, `readelf`/`objdump`/`nm`/`gdb` workflows, a complete C ELF inspector, ELF security mitigations, and binary instrumentation/forensics.

## How to Use This Book

Read Chapters 1–12 sequentially if you want a complete mental model of linking and loading, or jump directly to individual chapters as a desk reference when debugging a linker script, relocation error, or core dump. Every chapter includes exact C struct definitions from `<elf.h>` alongside runnable shell or C verification commands.

## Prerequisites

- Working fluency in C (pointers, structs, integer widths, bitwise masks) and basic x86-64 or AArch64 assembly reading skills.
- A Linux host or container with `gcc` or `clang`, GNU `binutils` (`readelf`, `objdump`, `nm`, `ld`), `gdb`, and `strace`.

## Conventions

- Standard `<elf.h>` types (`Elf64_Ehdr`, `Elf64_Shdr`, `Elf64_Phdr`, `Elf64_Sym`, `Elf64_Rela`, `Elf64_Dyn`) are shown with exact field names and byte offsets.
- Multi-byte hexadecimal values follow little-endian x86-64/AArch64 conventions unless explicitly noted as big-endian (`ELFDATA2MSB`).
- Shell labs use `$` for unprivileged commands and keep scratch artifacts inside `/tmp/elf-lab/`.
