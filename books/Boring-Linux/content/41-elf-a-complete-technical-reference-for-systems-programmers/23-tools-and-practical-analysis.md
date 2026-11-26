# Chapter 23: Tools and Practical Analysis

## `readelf`: Complete Option Reference and Interpretation

`readelf` parses ELF structures directly from disk without using BFD abstractions—making it the single most trustworthy ELF inspection tool:

| Command | Inspects |
| :--- | :--- |
| `readelf -h <bin>` | ELF header (`e_type`, `e_machine`, `e_entry`, table offsets) |
| `readelf -l -W <bin>` | Program Header Table (`PT_LOAD`, `PT_GNU_STACK`, `PT_GNU_RELRO`, section-to-segment map) |
| `readelf -S -W <bin>` | Section Header Table (flags, addresses, offsets, sizes) |
| `readelf -s -W <bin>` | Both `.dynsym` and `.symtab` symbol tables |
| `readelf -r -W <bin>` | All relocation tables (`.rela.dyn`, `.rela.plt`) |
| `readelf -d <bin>` | `.dynamic` entries (`DT_NEEDED`, `DT_RUNPATH`, `DT_FLAGS_1`) |
| `readelf -n <bin>` | Notes (`.note.gnu.build-id`, `GNU_PROPERTY`, ABI tag) |
| `readelf -V <bin>` | Symbol versioning (`Verdef` / `Verneed`) |

## `objdump`: Disassembly, Relocation Listing and Headers

- `objdump -d -M intel <bin>`: Disassemble executable sections in Intel syntax.
- `objdump -dr <file.o>`: Disassemble with inline relocation annotations beside every patched instruction.
- `objdump -s -j .rodata <bin>`: Hex + ASCII dump of a specific section.

## `nm`: Symbol Table Inspection and Symbol Types

`nm -C --defined-only <bin>` demangles C++ symbols and prints single-letter codes: `T`/`t` (global/local text), `D`/`d` (initialized data), `B`/`b` (BSS), `R`/`r` (rodata), `U` (undefined external), `W`/`w` (weak). Use `nm -D` for stripped binaries (reads `.dynsym`).

## `ldd`: Dynamic Dependency Analysis

`ldd <bin>` prints resolved shared library paths by running the target's dynamic linker with `LD_TRACE_LOADED_OBJECTS=1`.

> [!WARNING]
> **Never run `ldd` on an untrusted binary!** Some `ldd` implementations execute the target binary directly if its `PT_INTERP` is crafted maliciously. Always use **`readelf -d <bin> | grep NEEDED`** or **`objdump -p <bin> | grep NEEDED`** for safe static inspection of untrusted files.

## `file`, `hexdump` and `strace` for Binary Analysis

- `file <bin>`: Quick summary of class, endianness, PIE/dynamic status, interpreter, Build ID, and stripped state.
- `hexdump -C -s <off> -n <len> <bin>`: Raw byte verification at exact `sh_offset`/`p_offset`.
- `strace -e trace=execve,openat,mmap,mprotect ./app`: Watch `ld.so` map `.so` files and apply `mprotect(PROT_READ)` for RELRO live.

## GDB for Runtime ELF Inspection: Breakpoints, Memory Maps and Symbols

Inside `gdb ./app`:
- `info files`: Lists every mapped section of the executable and loaded libraries.
- `info proc mappings`: Shows live `/proc/<pid>/maps` virtual address ranges and `r-xp`/`rw-p` permissions.
- `info auxv`: Displays the kernel's Auxiliary Vector (`AT_ENTRY`, `AT_RANDOM`).
- `x/8gx &_GLOBAL_OFFSET_TABLE_`: Examines live GOT entries before and after a PLT call.
