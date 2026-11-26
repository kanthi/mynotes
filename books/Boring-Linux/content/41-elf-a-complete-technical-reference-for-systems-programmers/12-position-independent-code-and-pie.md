# Chapter 12: Position-Independent Code and PIE

## Position-Independent Code: Concepts and Requirements

**Position-Independent Code (PIC)** is machine code that executes correctly regardless of which virtual base address it is mapped to—without modifying a single byte of `.text`. Because `.text` stays clean and unmodified, 50 processes running on a Linux server can share the exact same physical RAM pages of `libc.so.6` and `libssl.so.3` simultaneously.

## PIC vs PIE: Shared Objects and Position-Independent Executables

| Property | PIC (`-fPIC -shared`) | PIE (`-fPIE -pie`) |
| :--- | :--- | :--- |
| **Output `e_type`** | `ET_DYN` (shared library `.so`) | `ET_DYN` (executable with `PT_INTERP` + `DF_1_PIE`) |
| **Symbol preemption** | Global symbols may be interposed (uses GOT/PLT) | Main executable symbols cannot be preempted (faster PC-relative access) |
| **Primary goal** | Share physical `.text` pages across processes | Randomize main executable base address via ASLR |

## Relative Addressing and GOT/PCREL Relocations

On x86-64 and AArch64, PIC/PIE is nearly free because both ISAs natively support **PC-relative addressing**:
- Local data and hidden functions use direct `lea msg(%rip), %rdi` (`R_X86_64_PC32`)—no GOT lookup required.
- External data accesses load the address from a GOT slot via `mov var@GOTPCREL(%rip), %rax`.
- Static pointer tables in `.data` emit `R_X86_64_RELATIVE` entries so `ld.so` adds the randomized load base `B` at startup.

## PIE and ASLR: Security Benefits and Implementation

Without PIE, an `ET_EXEC` binary always loads at a fixed virtual address (e.g., `0x400000`), handing attackers deterministic ROP gadget addresses even when the stack, heap, and `libc` are randomized. With **PIE + kernel ASLR** (`/proc/sys/kernel/randomize_va_space = 2`), every execution maps the main binary to a fresh random virtual address.

## Building PIE: Compiler Flags, Linker Options and Resulting ELF

```bash
# Modern GCC/Clang enable PIE by default; compare explicitly:
cat > /tmp/pie_lab.c <<'C'
#include <stdio.h>
int main(void) { printf("main @ %p
", (void *)main); return 0; }
C

cc -fPIE -pie /tmp/pie_lab.c -o /tmp/pie_bin
cc -fno-pie -no-pie /tmp/pie_lab.c -o /tmp/nopie_bin

readelf -h /tmp/pie_bin   | grep Type
readelf -h /tmp/nopie_bin | grep Type
/tmp/pie_bin; /tmp/pie_bin
rm -f /tmp/pie_lab.c /tmp/pie_bin /tmp/nopie_bin
```

Notice how `/tmp/pie_bin` prints a different `main` address on every run while `/tmp/nopie_bin` stays pinned at `0x401...`.
