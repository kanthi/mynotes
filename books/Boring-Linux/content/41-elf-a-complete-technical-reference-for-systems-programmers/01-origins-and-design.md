# Chapter 1: Origins and Design

## The Need for a Unified Format

Before ELF, Unix binary formats fractured along historical lines. Early Bell Labs Unix used **`a.out`** (assembler output), a minimalist layout with fixed `.text`, `.data`, and `.bss` regions that had no clean support for arbitrary custom sections, shared libraries, or debugging metadata. System V Release 3 introduced **COFF (Common Object File Format)**, which added a section table—but COFF's fixed-size 8-character section names, rigid relocation encoding, and lack of standardized dynamic linking forced every vendor to invent incompatible extensions. Systems programmers needed a single, extensible binary container that worked uniformly for relocatable object files, executables, shared libraries, and core dumps across 32-bit and 64-bit CPUs of either endianness.

## Design Objectives: Portability, Extensibility and Separation of Concerns

ELF succeeded because its designers baked four architectural principles into the specification:

1. **Portability across architectures and word sizes:** A 16-byte machine-independent identification header (`e_ident`) declares 32-bit vs. 64-bit class and little-endian vs. big-endian byte order before any multi-byte integer is parsed.
2. **Extensibility without breaking parsers:** Section types (`SHT_*`), segment types (`PT_*`), dynamic tags (`DT_*`), and note types (`NT_*`) reserve explicit numeric ranges for OS-specific (`LOOS..HIOS`) and processor-specific (`LOPROC..HIPROC`) extensions.
3. **Separation of linking and execution concerns:** Linkers care about granular logical **sections** (`.text`, `.rodata`, `.symtab`, `.rela.text`); the OS kernel cares only about page-aligned memory **segments** (`PT_LOAD`) with `R/W/X` permissions. ELF represents both views cleanly in one file.
4. **Zero-copy memory mapping:** File offsets and virtual addresses are congruent modulo the system page size (`p_offset % page_size == p_vaddr % page_size`), letting the kernel demand-page executable code directly from disk via `mmap`.

## The SVR4 Project and ELF Standardization

ELF was developed by Unix System Laboratories (USL) as part of **System V Release 4 (SVR4)** in 1988 and published in the System V Application Binary Interface (SysV ABI). The **Tool Interface Standards (TIS)** committee standardized ELF v1.2 in 1995, and the Linux Standard Base (LSB) plus the generic ABI (`gABI`) and processor-specific ABI (`psABI`) working groups continue to evolve ELF for modern hardware (x86-64, AArch64, RISC-V, LoongArch).

## ELF in the Modern Ecosystem

Today ELF is the native executable format for Linux, FreeBSD, OpenBSD, NetBSD, Solaris/illumos, Android (native JNI/NDK), PlayStation, Nintendo Switch homebrew, and bare-metal embedded firmware (ARM Cortex-M, RISC-V `.elf` images converted to `.bin`/`.hex` for flashing). Even Linux eBPF programs are compiled by Clang/LLVM into ELF relocatable object files (`EM_BPF`) before `libbpf` loads them into the kernel.

## Terminology and Conventions Used in This Book

| Term | Exact Meaning in ELF |
| :--- | :--- |
| **Object file** | Any ELF file: relocatable (`.o`), executable, shared object (`.so`), or core dump |
| **Section** | Contiguous byte range described by an `Elf64_Shdr`; primary unit of static linking |
| **Segment** | Contiguous memory mapping described by an `Elf64_Phdr`; primary unit of loading |
| **VMA / `vaddr`** | Virtual Memory Address where a section or segment resides in a running process |
| **LMA / `paddr`** | Load/Physical Memory Address (used mainly in embedded/firmware linker scripts) |
| **ABI** | Application Binary Interface: calling conventions, register roles, relocation semantics |

## Hands-On Lab: Inspect ELF Identity on Your Host

```bash
# Inspect ELF headers of /bin/ls (or any local ELF binary on Linux)
file /bin/ls
readelf -h /bin/ls 2>/dev/null | head -n 20 || true
```
