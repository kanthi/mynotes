# Chapter 19: Notes and Metadata

## Note Section Structure: `n_name`, `n_descsz`, `n_desc`

ELF notes (`SHT_NOTE` / `PT_NOTE`) carry structured, self-describing metadata such as build IDs, minimum kernel ABI versions, hardware feature flags, and core dump register snapshots. Each note starts with a 12-byte header followed by 4-byte-aligned `name` and `desc` payloads:

```c
typedef struct {
    uint32_t n_namesz; /* Byte length of owner name (including NUL) */
    uint32_t n_descsz; /* Byte length of descriptor payload */
    uint32_t n_type;   /* Owner-specific note type (NT_*) */
} Elf64_Nhdr;          /* Followed by align4(name) + align4(desc) */
```

## `PT_NOTE` Segments and Core Dump Notes

In executables and `.so` files, `PT_NOTE` points to allocatable `.note.*` sections. In `ET_CORE` dumps, the kernel emits a `PT_NOTE` segment containing thread registers, process metadata, and mapped file paths so GDB can reconstruct the exact crash state without the original live process.

## GNU Build Attributes: `.note.gnu.build-id`

Linked automatically via `--build-id` (SHA-1 or xxHash), `.note.gnu.build-id` (`n_name = "GNU"`, `n_type = NT_GNU_BUILD_ID (3)`) embeds a cryptographic fingerprint uniquely identifying the exact binary build.

## Build ID: Computing and Using ELF Identity

Debuggers (`gdb`, `lldb`), profilers (`perf`), and `debuginfod` servers look up detached `.debug` symbol files purely by Build ID (`/usr/lib/debug/.build-id/ab/cdef...debug`) rather than fragile filenames—ensuring symbols always match the exact crashed binary.

## Core Dump Notes: `NT_PRSTATUS`, `NT_PRPSINFO`, `NT_FILE`, `NT_SIGINFO`

| Note Type | Value | Contents in `ET_CORE` |
| :--- | :--- | :--- |
| `NT_PRSTATUS` | `1` | Per-thread CPU general-purpose registers (`pt_regs`), PID, signal number |
| `NT_PRPSINFO` | `3` | Process name, UID/GID, command-line arguments (`prpsinfo`) |
| `NT_AUXV` | `6` | Copy of the process's original Auxiliary Vector |
| `NT_SIGINFO` | `0x53494749` | Exact `siginfo_t` faulting address (`si_addr`) |
| `NT_FILE` | `0x46494c45` | Table of all memory-mapped file paths and virtual address ranges |

## Notes in Shared Objects and Executables

Inspect all embedded notes—including `.note.ABI-tag` (minimum Linux kernel version), `.note.gnu.build-id`, and `.note.gnu.property`—with `readelf -n`:

```bash
readelf -n /bin/ls 2>/dev/null || true
```
