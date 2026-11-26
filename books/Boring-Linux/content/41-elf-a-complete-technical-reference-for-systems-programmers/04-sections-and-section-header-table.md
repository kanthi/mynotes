# Chapter 4: Sections and the Section Header Table

## The Section Header Table Structure and Layout

The **Section Header Table (SHT)** is an array of `e_shnum` contiguous `Elf64_Shdr` entries (64 bytes each) starting at file offset `e_shoff`. Every section in a relocatable `.o` file or unstripped executable has exactly one header entry describing its name, type, permissions, file offset, and size.

```c
typedef struct {
    uint32_t sh_name;      /* Byte offset into .shstrtab */
    uint32_t sh_type;      /* Section semantics (SHT_*) */
    uint64_t sh_flags;     /* Bitmask (SHF_WRITE, SHF_ALLOC, ...) */
    uint64_t sh_addr;      /* Virtual address in memory (0 if non-alloc) */
    uint64_t sh_offset;    /* File offset in bytes */
    uint64_t sh_size;      /* Size in bytes (even for SHT_NOBITS!) */
    uint32_t sh_link;      /* Associated section index (e.g. .strtab) */
    uint32_t sh_info;      /* Extra type-dependent metadata */
    uint64_t sh_addralign; /* Required alignment (power of 2) */
    uint64_t sh_entsize;   /* Entry size for fixed-record tables */
} Elf64_Shdr;              /* Total: 64 bytes */
```

## `sh_name`, `sh_type` and All Defined Section Types

| Constant | Value | Meaning |
| :--- | :--- | :--- |
| `SHT_NULL` | `0` | Inactive sentinel header (always index 0) |
| `SHT_PROGBITS` | `1` | Program-defined data or machine code (`.text`, `.data`, `.rodata`) |
| `SHT_SYMTAB` | `2` | Full static linker symbol table (`.symtab`) |
| `SHT_STRTAB` | `3` | Null-terminated string table (`.strtab`, `.shstrtab`, `.dynstr`) |
| `SHT_RELA` | `4` | Relocation entries with explicit addends (`.rela.*`) |
| `SHT_HASH` | `5` | System V symbol hash table (`.hash`) |
| `SHT_DYNAMIC` | `6` | Dynamic linking array (`.dynamic`) |
| `SHT_NOTE` | `7` | Structured vendor/build metadata (`.note.*`) |
| `SHT_NOBITS` | `8` | Occupies **zero** bytes on disk, zero-filled in RAM (`.bss`, `.tbss`) |
| `SHT_REL` | `9` | Relocation entries without explicit addends |
| `SHT_DYNSYM` | `11` | Minimal runtime dynamic symbol table (`.dynsym`) |
| `SHT_INIT_ARRAY` | `14` | Array of constructor function pointers (`.init_array`) |
| `SHT_FINI_ARRAY` | `15` | Array of destructor function pointers (`.fini_array`) |
| `SHT_GNU_HASH` | `0x6ffffff6` | GNU Bloom-filtered symbol hash table (`.gnu.hash`) |
| `SHT_GNU_verdef/need` | `0x6ffffffd/e` | Symbol versioning tables |

## Section Flags: `SHF_WRITE`, `SHF_ALLOC`, `SHF_EXECINSTR` and Beyond

- `SHF_WRITE` (`0x1`): Writable during process execution (`.data`, `.bss`, `.got`).
- `SHF_ALLOC` (`0x2`): Occupies memory in the running process image (absent on `.symtab`, `.comment`, `.debug_*`).
- `SHF_EXECINSTR` (`0x4`): Contains executable CPU instructions (`.text`, `.plt`, `.init`).
- `SHF_MERGE` (`0x10`) & `SHF_STRINGS` (`0x20`): Linker may deduplicate identical constants/strings across `.o` files.
- `SHF_TLS` (`0x400`): Holds Thread-Local Storage templates (`.tdata`, `.tbss`).

## Address, Offset, Size, Link, Info and Alignment Fields

`sh_link` and `sh_info` connect related tables without hardcoding names:
- For `SHT_SYMTAB` / `SHT_DYNSYM`: `sh_link` points to the associated string table (`.strtab` / `.dynstr`), and `sh_info` stores **1 + the index of the last `STB_LOCAL` symbol**.
- For `SHT_RELA` / `SHT_REL`: `sh_link` points to the symbol table, and `sh_info` points to the target section being patched (e.g., `.text`).

## Section Name String Table and the Null Section

Section index `0` (`SHN_UNDEF`) is always a zeroed `SHT_NULL` sentinel. Section names are resolved by taking `sh_name` as a byte offset into the section indexed by `e_shstrndx` (typically `.shstrtab`).

## Special Sections: `.text`, `.data`, `.bss`, `.rodata`, `.symtab`, `.strtab`

- **`.text`** (`PROGBITS`, `AX`): Compiled machine code.
- **`.rodata`** (`PROGBITS`, `A`): Immutable constants, string literals, jump tables.
- **`.data`** (`PROGBITS`, `WA`): Initialized mutable global and static variables.
- **`.bss`** (`NOBITS`, `WA`): Zero-initialized globals; consumes zero disk space.
- **`.symtab` / `.strtab`** (non-`ALLOC`): Complete symbol names and addresses; stripped by `strip` in release builds without affecting runtime execution.

## Hands-On Lab: Inspect Section Flags and `.bss` Zero-Disk Storage

```bash
readelf -S -W /bin/ls 2>/dev/null | head -n 35 || true
```
