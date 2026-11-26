# Chapter 21: Architecture-Specific Details: x86-64

## x86-64 ELF Conventions: ABI and Calling Convention Impact

The System V AMD64 psABI governs every x86-64 Linux ELF binary:
- **Integer/pointer arguments:** `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`; return value in `rax` (and `rdx`).
- **Floating-point/SIMD arguments:** `xmm0`–`xmm7`.
- **Stack alignment:** `%rsp` must be **16-byte aligned** before `call` (so `%rsp % 16 == 8` on function entry).
- **Red zone:** 128 bytes below `%rsp` available to leaf functions without adjusting `%rsp` (disabled in kernel code via `-mno-red-zone`).
- **Code models:** Default `-mcmodel=small` assumes code and static data fit within +/-2 GiB, allowing compact 32-bit RIP-relative displacements.

## x86-64 Relocation Types: Complete Reference

| Relocation | ID | Width | Formula | Role |
| :--- | :--- | :--- | :--- | :--- |
| `R_X86_64_64` | `1` | 64-bit | `S + A` | Direct 64-bit pointer |
| `R_X86_64_PC32` | `2` | 32-bit | `S + A - P` | RIP-relative data/branch |
| `R_X86_64_PLT32` | `4` | 32-bit | `L + A - P` | Function call (relaxed to PC32 if local) |
| `R_X86_64_GLOB_DAT` | `6` | 64-bit | `S` | GOT data/fn slot |
| `R_X86_64_JUMP_SLOT` | `7` | 64-bit | `S` | `.got.plt` function slot |
| `R_X86_64_RELATIVE` | `8` | 64-bit | `B + A` | PIE / `.so` base-relative pointer |
| `R_X86_64_GOTPCREL` | `9` | 32-bit | `G + GOT + A - P` | PC-relative GOT slot offset |
| `R_X86_64_TPOFF32` | `23` | 32-bit | `TPOFF(S) + A` | Local-Exec TLS offset from `%fs:0` |
| `R_X86_64_IRELATIVE` | `37` | 64-bit | `indirect(B + A)` | IFUNC load-time resolver |
| `R_X86_64_REX_GOTPCRELX` | `42` | 32-bit | Relaxable GOT load | Linker converts `mov` → `lea` when local |

## x86-64 PLT/GOT Layout and Calling Sequences

With Intel CET (`-fcf-protection=full`), GCC emits a split `.plt.sec` (containing `endbr64; bnd jmp *GOT(%rip)`) and `.plt` (holding the lazy fallback push/jmp stubs). Without CET, a standard 16-byte PLT entry is `jmp *GOT(%rip); pushq $reloc_idx; jmp PLT[0]`.

## x86-64 TLS: `FS` Register and Access Models

Linux configures the `%fs` segment base via `arch_prctl(ARCH_SET_FS, tcb_addr)`. Local-Exec accesses emit `mov %fs:-0x8, %rax`, while Initial-Exec loads the offset from GOT (`mov var@GOTTPOFF(%rip), %rax; mov %fs:(%rax), %eax`).

## x86-64 `e_flags` and Machine-Specific Behavior

`e_flags` is `0` on x86-64; ISA microarchitecture levels (`x86-64-v2`, `x86-64-v3` AVX2, `x86-64-v4` AVX-512) are recorded instead in `.note.gnu.property` (`GNU_PROPERTY_X86_ISA_1_NEEDED`).

## Hands-On Lab: Inspect RIP-Relative Addressing and `endbr64`

```bash
objdump -d /bin/true 2>/dev/null | grep -E 'endbr64|%rip' | head -n 15 || true
```
