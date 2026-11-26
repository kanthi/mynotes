# Chapter 20: GNU and Linux Extensions

## GNU Hash Table: `DT_GNU_HASH` Details

`DT_GNU_HASH` (`0x6ffffef5`) begins with four 32-bit header words: `nbuckets`, `symoffset`, `bloom_size`, and `bloom_shift`, followed by the 64-bit Bloom filter array, bucket array, and hash chains. Because `symoffset` separates unhashed local/undefined entries from hashed exported entries, `ld.so` never wastes buckets on imported symbols.

## RELRO: `PT_GNU_RELRO` and Partial RELRO vs Full RELRO

- **Partial RELRO (`-Wl,-z,relro`):** Reorders `.dynamic`, `.init_array`, `.fini_array`, and `.got` *before* `.data`/`.bss` and marks them read-only after startup. However, `.got.plt` remains writable for lazy binding.
- **Full RELRO (`-Wl,-z,relro,-z,now`):** Combines `PT_GNU_RELRO` with `DT_BIND_NOW`. Every external function resolves eagerly at startup and `.got.plt` is folded into the read-only RELRO page—completely closing GOT overwrite attacks.

## IFUNC: Indirect Functions

`STT_GNU_IFUNC` (`__attribute__((ifunc("resolver")))`) allows a library (notably glibc's `memcpy`, `strlen`, and `strcmp`) to ship multiple CPU-optimized implementations (SSE2, AVX2, AVX-512) under one symbol name. At load time, `ld.so` invokes the resolver once via `R_X86_64_IRELATIVE`, inspects CPUID flags, and writes the winning function address straight into the GOT.

## `Verneed` and `Verdef`: Symbol Versioning

Shared libraries maintain backward ABI compatibility across decades using `.gnu.version` (`SHT_GNU_versym`), `.gnu.version_d` (`Verdef`, exported versions like `GLIBC_2.38`), and `.gnu.version_r` (`Verneed`, required versions imported by an executable). A single `libc.so.6` can simultaneously export `memcpy@GLIBC_2.2.5` for legacy binaries and `memcpy@@GLIBC_2.14` as the default for new builds.

## Stack Protector and Canary Implementation

Compiling with `-fstack-protector-strong` inserts a secret per-process guard value loaded from the Thread Control Block (`%fs:0x28` on x86-64, initialized from `AT_RANDOM` with a leading NUL byte to block C string leaks) into the stack frame prologue and verifies it before `ret`—calling `__stack_chk_fail@plt` on mismatch.

## GNU Property Notes: `.note.gnu.property`

`PT_GNU_PROPERTY` advertises hardware Control-Flow Integrity capabilities—`GNU_PROPERTY_X86_FEATURE_1_IBT` (Indirect Branch Tracking `endbr64`), `SHSTK` (Intel Shadow Stack), and `GNU_PROPERTY_AARCH64_FEATURE_1_BTI`/`PAC`. The kernel enables hardware enforcement only when the executable *and every loaded `.so`* declare support.

## Hands-On Lab: Audit GNU Extensions on a System Binary

```bash
readelf -V /bin/ls 2>/dev/null | head -n 25 || true
readelf -n /bin/ls 2>/dev/null | grep -A4 'GNU_PROPERTY' || true
```
