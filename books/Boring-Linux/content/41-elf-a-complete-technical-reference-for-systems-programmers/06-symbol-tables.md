# Chapter 6: Symbol Tables

## Symbol Table Structure: `Elf64_Sym` Fields and Semantics

Symbol tables (`.symtab` for static linking/debugging and `.dynsym` for dynamic runtime linking) map human-readable names to virtual addresses, sizes, and linkage properties.

```c
typedef struct {
    uint32_t      st_name;  /* Byte offset into linked string table */
    unsigned char st_info;  /* Upper 4 bits = binding, lower 4 bits = type */
    unsigned char st_other; /* Lower 2 bits = visibility (STV_*) */
    uint16_t      st_shndx; /* Defining section index (or SHN_UNDEF/ABS/COMMON) */
    uint64_t      st_value; /* Symbol address (or offset in ET_REL) */
    uint64_t      st_size;  /* Object/function byte size (0 if unknown) */
} Elf64_Sym;                /* Total: 24 bytes */

#define ELF64_ST_BIND(i)    ((i) >> 4)
#define ELF64_ST_TYPE(i)    ((i) & 0xf)
#define ELF64_ST_VISIBILITY(o) ((o) & 0x3)
```

## Symbol Names and the String Table Relationship

`st_name` is an integer byte offset into the string table referenced by the symbol table section's `sh_link` (`.strtab` for `.symtab`; `.dynstr` for `.dynsym`). An offset of `0` denotes an unnamed symbol (such as section symbols `STT_SECTION`).

## Symbol Types: `STT_NOTYPE`, `STT_FUNC`, `STT_OBJECT` and Complete List

| Constant | Value | Meaning |
| :--- | :--- | :--- |
| `STT_NOTYPE` | `0` | Unspecified type (common for assembly labels) |
| `STT_OBJECT` | `1` | Data object (global/static variable) |
| `STT_FUNC` | `2` | Executable function |
| `STT_SECTION` | `3` | Section anchor used by relocations |
| `STT_FILE` | `4` | Source filename marker (`STB_LOCAL`, `SHN_ABS`) |
| `STT_COMMON` | `5` | Uninitialized common block |
| `STT_TLS` | `6` | Thread-local variable (offset within TLS block) |
| `STT_GNU_IFUNC` | `10` | Indirect function resolved at load time to CPU-tuned implementation |

## Symbol Binding: Local, Global, Weak and Their Linking Implications

- **`STB_LOCAL` (`0`):** Invisible outside the defining `.o` file (`static` in C). Multiple files can define identical local names without collision.
- **`STB_GLOBAL` (`1`):** Visible across all linked object files; two strong global definitions of the same name cause a `multiple definition` linker error.
- **`STB_WEAK` (`2`):** Overridden silently if any `STB_GLOBAL` definition exists; if completely undefined at link time, resolves to `0` instead of failing (enabling optional hooks).

## Symbol Visibility: `STV_DEFAULT`, `STV_HIDDEN`, `STV_PROTECTED`, `STV_INTERNAL`

- **`STV_DEFAULT` (`0`):** Exported in `.dynsym` and subject to runtime symbol interposition (`LD_PRELOAD`).
- **`STV_HIDDEN` (`2`):** Invisible outside the current shared object—shrinks `.dynsym`, eliminates PLT overhead, and prevents accidental ABI export (`-fvisibility=hidden`).
- **`STV_PROTECTED` (`3`):** Exported to external callers, but internal references inside the defining `.so` cannot be preempted.
- **`STV_INTERNAL` (`1`):** Processor-specific stricter hidden form.

## Symbol Table Organization and Index Conventions

Index `0` (`STN_UNDEF`) is always a zeroed null symbol. ELF requires **all `STB_LOCAL` symbols to appear before all `STB_GLOBAL`/`STB_WEAK` symbols**, and stores the first non-local index in `sh_info` so linkers can skip local symbols in $O(1)$.

## Hands-On Lab: Compare `.dynsym` vs. `.symtab` Visibility

```bash
cat > /tmp/sym_demo.c <<'C'
static int internal_helper(void) { return 1; }
__attribute__((visibility("hidden"))) int hidden_api(void) { return internal_helper() + 1; }
__attribute__((weak)) int overridable_hook(void) { return 42; }
int public_api(void) { return hidden_api() + overridable_hook(); }
C
cc -shared -fPIC /tmp/sym_demo.c -o /tmp/libsym_demo.so
readelf --syms -W /tmp/libsym_demo.so | grep -E 'helper|hidden_api|overridable|public_api'
rm -f /tmp/sym_demo.c /tmp/libsym_demo.so
```
