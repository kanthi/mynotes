# Chapter 5: Segments and the Program Header Table

## The Program Header Table Structure and Layout

While linkers organize bytes into dozens of sections, the Linux kernel maps an executable into virtual memory using only the **Program Header Table (PHT)**—an array of `e_phnum` `Elf64_Phdr` entries (56 bytes each) at offset `e_phoff`.

```c
typedef struct {
    uint32_t p_type;   /* Segment type (PT_LOAD, PT_DYNAMIC, ...) */
    uint32_t p_flags;  /* Permissions bitmask (PF_R, PF_W, PF_X) */
    uint64_t p_offset; /* Byte offset in the ELF file */
    uint64_t p_vaddr;  /* Target virtual address in process memory */
    uint64_t p_paddr;  /* Physical address (mostly embedded/firmware) */
    uint64_t p_filesz; /* Number of bytes copied/mapped from file */
    uint64_t p_memsz;  /* Number of bytes occupied in RAM (>= p_filesz) */
    uint64_t p_align;  /* Page alignment constraint (e.g., 0x1000) */
} Elf64_Phdr;          /* Total: 56 bytes */
```

## Segment Types: `PT_LOAD`, `PT_DYNAMIC`, `PT_INTERP`, `PT_GNU_STACK` and All Others

| Constant | Value | Runtime Role |
| :--- | :--- | :--- |
| `PT_NULL` | `0` | Unused entry |
| `PT_LOAD` | `1` | Loadable memory segment mapped via `mmap` |
| `PT_DYNAMIC` | `2` | Points to `.dynamic` table for `ld.so` |
| `PT_INTERP` | `3` | Null-terminated path to dynamic linker (e.g. `/lib64/ld-linux-x86-64.so.2`) |
| `PT_NOTE` | `4` | Auxiliary build-ID, ABI tag, or core dump notes |
| `PT_PHDR` | `6` | Location and size of the Program Header Table itself in memory |
| `PT_TLS` | `7` | Thread-Local Storage template (`.tdata` + `.tbss`) |
| `PT_GNU_EH_FRAME` | `0x6474e550` | Sorted DWARF exception-unwinding lookup table |
| `PT_GNU_STACK` | `0x6474e551` | Declares stack permissions (`RW` = NX enforced; `RWE` = executable stack) |
| `PT_GNU_RELRO` | `0x6474e552` | Region `ld.so` marks read-only via `mprotect` after relocations |
| `PT_GNU_PROPERTY` | `0x6474e553` | Hardware security features (Intel CET IBT/SHSTK, ARM BTI/PAC) |

## The Segment Type System: Standard and GNU Extensions

Standard `PT_*` values occupy `0..7`, while OS-specific extensions live in `0x60000000..0x6fffffff`. Modern Linux toolchains emit four separate `PT_LOAD` segments by default (`-z separate-code`): read-only headers/metadata (`R`), executable code (`R E`), read-only data (`R`), and read-write data/BSS (`R W`)—strictly enforcing **W^X** at page granularity.

## Memory Mapping Semantics: Offsets, Virtual Addresses, Sizes

When `p_memsz > p_filesz` (which happens on the writable `PT_LOAD` segment containing `.data` followed by `.bss`), the kernel maps `p_filesz` bytes from the file and zero-fills the remaining `p_memsz - p_filesz` bytes in anonymous RAM. Because hardware MMUs map whole pages, ELF mandates `p_offset % p_align == p_vaddr % p_align`.

## Segment Permissions: `PF_R`, `PF_W`, `PF_X` and NX Interaction

- `PF_X` (`0x1`) → `PROT_EXEC`
- `PF_W` (`0x2`) → `PROT_WRITE`
- `PF_R` (`0x4`) → `PROT_READ`

Hardware NX/XD bits in x86-64 and AArch64 page tables enforce non-execution on any segment lacking `PF_X`, including the stack whenever `PT_GNU_STACK` has `p_flags == PF_R | PF_W`.

## Overlapping Segments and File-to-Memory Mapping

Non-loadable segments (`PT_DYNAMIC`, `PT_INTERP`, `PT_NOTE`, `PT_GNU_RELRO`) almost always **overlap** a surrounding `PT_LOAD` segment—they act as typed pointers into bytes already mapped by `PT_LOAD` rather than creating duplicate mappings.

## Hands-On Lab: Map Sections to Segments with `readelf -l`

```bash
readelf -l -W /bin/ls 2>/dev/null | head -n 45 || true
```
