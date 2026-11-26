# Chapter 9: The Dynamic Section and Dynamic Linking

## Dynamic Section Structure: `Elf64_Dyn` and `DT_*` Entries

Because the kernel and dynamic linker (`ld.so`) do not require a Section Header Table at runtime, all metadata needed to link an executable or shared object dynamically is packed into a single `PT_DYNAMIC` segment pointing to the `.dynamic` array of `Elf64_Dyn` structs, terminated by a `DT_NULL` (`0`) entry.

```c
typedef struct {
    int64_t d_tag; /* Tag identifying interpretation (DT_*) */
    union {
        uint64_t d_val; /* Integer value or string-table offset */
        uint64_t d_ptr; /* Program virtual address */
    } d_un;
} Elf64_Dyn;       /* Total: 16 bytes */
```

## `DT_NEEDED`: Dependency Declaration and Resolution

Each `DT_NEEDED` entry stores an offset into `.dynstr` naming a required shared library (for example, `libc.so.6` or `libm.so.6`). `ld.so` performs a breadth-first traversal of `DT_NEEDED` entries across the executable and every loaded library to build the global symbol lookup scope.

## `DT_STRTAB`, `DT_SYMTAB`, `DT_HASH`: The Dynamic Name System

- **`DT_STRTAB` & `DT_STRSZ`:** Virtual address and byte size of `.dynstr`.
- **`DT_SYMTAB` & `DT_SYMENT`:** Virtual address of `.dynsym` and stride (`24` bytes for ELF64).
- **`DT_HASH` / `DT_GNU_HASH`:** Hash tables enabling fast $O(1)$ dynamic symbol lookup.

## `DT_RELA`, `DT_REL`: Dynamic Relocation Descriptors

- **`DT_RELA`, `DT_RELASZ`, `DT_RELAENT`:** Describe `.rela.dyn` (data and GOT relocations).
- **`DT_JMPREL`, `DT_PLTRELSZ`, `DT_PLTREL`:** Describe `.rela.plt` (function `JUMP_SLOT` relocations).
- **`DT_RELR`, `DT_RELRSZ`, `DT_RELRENT`:** Compact bitmask encoding for relative relocations (`-z pack-relative-relocs`).

## `DT_RPATH`, `DT_RUNPATH`, `DT_ORIGIN`: Library Search Paths

- **`DT_RPATH`** (deprecated, searched **before** `LD_LIBRARY_PATH`): Hardcoded colon-separated search path.
- **`DT_RUNPATH`** (modern default via `--enable-new-dtags`, searched **after** `LD_LIBRARY_PATH`): Allows operators to override libraries cleanly without binary patching.
- **`$ORIGIN`:** Special token expanded by `ld.so` to the directory containing the ELF binary itself (e.g., `-Wl,-rpath,'$ORIGIN/../lib'`), enabling relocatable application bundles.

## `DT_SONAME`, `DT_FLAGS`, `DT_FLAGS_1`: Identity and Capabilities

- **`DT_SONAME`:** Canonical ABI name embedded in a `.so` (e.g., `libssl.so.3`). When an app links against `libssl.so`, the linker records the library's `DT_SONAME` into the app's `DT_NEEDED`.
- **`DT_FLAGS` & `DT_FLAGS_1`:** Bitmasks carrying `DF_BIND_NOW`, `DF_1_NOW` (Full RELRO), `DF_1_PIE` (distinguishing PIE executables from `.so` files), and `DF_1_NODELETE`.

## Hands-On Lab: Inspect `.dynamic` Entries with `readelf -d`

```bash
readelf -d /bin/ls 2>/dev/null | head -n 30 || true
```
