# Chapter 22: Architecture-Specific Details: AArch64

## AArch64 ELF Conventions: AAPCS64 and Calling Convention Impact

64-bit ARM (AArch64, `EM_AARCH64 = 183`) uses fixed-width 32-bit instructions and follows **AAPCS64**:
- **Arguments & return:** `x0`–`x7` (integers/pointers) and `v0`–`v7` (SIMD/FP); return value in `x0`–`x1`.
- **Link register (`x30` / `lr`):** `bl target` stores the return address in `x30` rather than pushing it onto the stack.
- **Stack pointer (`sp`):** Hardware enforces strict **16-byte alignment** on every memory access using `sp`.
- **Page-relative addressing:** Because a 32-bit instruction cannot embed a 64-bit address, AArch64 pairs `adrp x0, page` (21-bit PC-relative 4 KiB page offset, +/-4 GiB reach) with `add x0, x0, #:lo12:sym` (12-bit page offset).

## AArch64 Relocation Types: Key Types and Semantics

| Relocation | ID | Role |
| :--- | :--- | :--- |
| `R_AARCH64_ABS64` | `257` | 64-bit absolute pointer (`S + A`) |
| `R_AARCH64_ADR_PREL_PG_HI21` | `275` | Patches 21-bit page delta into `adrp` |
| `R_AARCH64_ADD_ABS_LO12_NC` | `277` | Patches 12-bit offset into `add` |
| `R_AARCH64_CALL26` / `JUMP26` | `283`/`282` | 26-bit signed instruction offset (+/-128 MiB `bl`/`b`) |
| `R_AARCH64_ADR_GOT_PAGE` | `311` | `adrp` to GOT page |
| `R_AARCH64_LD64_GOT_LO12_NC` | `312` | `ldr` 8-byte GOT slot within page |
| `R_AARCH64_GLOB_DAT` / `JUMP_SLOT` | `1025`/`1026` | Dynamic GOT / PLT resolution |
| `R_AARCH64_RELATIVE` | `1027` | Load-base adjustment (`B + A`) |
| `R_AARCH64_TLSDESC` | `1031` | Default AArch64 dynamic TLS descriptor |

## AArch64 PLT/GOT Layout and Calling Sequences

Each AArch64 PLT stub uses scratch registers `x16` (`ip0`) and `x17` (`ip1`):
```asm
adrp  x16, Page(GOT[sym])
ldr   x17, [x16, #:lo12:GOT[sym]]
add   x16, x16, #:lo12:GOT[sym]
br    x17
```

## AArch64 TLS: `TPIDR_EL0` and Access Models

AArch64 uses **TLS Variant I**: `mrs x0, tpidr_el0` reads the thread pointer, and static TLS data sits at positive offsets above the 16-byte TCB. General-Dynamic uses **TLSDESC** by default for fast single-indirect resolution.

## AArch64 `e_flags` and Machine-Specific Behavior

`e_flags` is `0`; hardware security features **BTI** (Branch Target Identification) and **PAC** (Pointer Authentication `paciasp`/`autiasp`) are encoded in `.note.gnu.property`.

## Hands-On Lab: Cross-Inspect AArch64 Relocations (or Inspect Native ARM64)

```bash
# On ARM64 Linux or with aarch64-linux-gnu-gcc installed:
uname -m
command -v aarch64-linux-gnu-readelf && echo "Cross binutils available" || echo "Use readelf on ARM64 target"
```
