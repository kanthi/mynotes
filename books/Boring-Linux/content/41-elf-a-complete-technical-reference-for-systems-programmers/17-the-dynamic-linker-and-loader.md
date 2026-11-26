# Chapter 17: The Dynamic Linker and Loader

## `PT_INTERP` and the Dynamic Linker Entry

When the Linux kernel loads a dynamically linked ELF executable, it reads the null-terminated path inside `PT_INTERP` (typically `/lib64/ld-linux-x86-64.so.2` on x86-64 glibc, `/lib/ld-linux-aarch64.so.1` on AArch64, or `/lib/ld-musl-x86_64.so.1` on Alpine). Instead of jumping straight to the executable's `e_entry`, the kernel maps `ld.so` into the process address space and transfers initial CPU control to **`ld.so`'s entry point**.

## Loading Sequence: Map, Parse, Resolve, Initialize

1. **Self-bootstrap:** `ld.so` relocates its *own* GOT/relative entries without calling any external functions.
2. **Parse auxiliary vector (`auxv`):** Reads `AT_PHDR`, `AT_PHNUM`, and `AT_ENTRY` to locate the already-mapped main executable's `PT_DYNAMIC` segment.
3. **Map dependencies:** Walks `DT_NEEDED` recursively (plus `LD_PRELOAD`), `mmap()`ing every shared library's `PT_LOAD` segments.
4. **Apply relocations:** Processes `.rela.dyn`, `.relr.dyn`, and (if `DT_BIND_NOW`) `.rela.plt`, then applies `mprotect(PROT_READ)` to every `PT_GNU_RELRO` region.
5. **Run constructors & jump:** Executes `.preinit_array` and `.init_array` in topological order, then jumps to `AT_ENTRY` (`_start`).

## Library Search Path Resolution: `LD_LIBRARY_PATH`, `RPATH`, `RUNPATH`, System Paths

`ld.so` searches for each `DT_NEEDED` soname in strict precedence order:
1. `DT_RPATH` of the requesting object (ignored if `DT_RUNPATH` is present; skipped for `setuid` binaries).
2. `LD_LIBRARY_PATH` environment variable (ignored for `AT_SECURE` / `setuid` binaries).
3. `DT_RUNPATH` of the requesting object.
4. Precompiled cache `/etc/ld.so.cache` (maintained by `ldconfig`).
5. Default system directories (`/lib64`, `/usr/lib64`, `/lib`, `/usr/lib`).

## Dependency Graph Resolution and Load Order

Libraries are loaded breadth-first into a global linked list (`link_map`). Circular dependencies (`libA` ↔ `libB`) are permitted during mapping, though constructor ordering between mutually dependent libraries is undefined and should be avoided.

## Relocation Application at Load Time

Load-time relocations fix up every `R_*_RELATIVE`, `R_*_GLOB_DAT`, `R_*_IRELATIVE` (IFUNC), and eager `R_*_JUMP_SLOT` entry across all mapped `link_map` nodes.

## Lazy Resolution and PLT Trampolines in Action

You can watch `ld.so` search paths, map libraries, resolve symbols, and execute constructors live using the built-in `LD_DEBUG` diagnostic variable:

```bash
LD_DEBUG=libs,bindings /bin/true 2>&1 | head -n 30
```
