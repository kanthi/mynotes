# Chapter 14: Constructors, Destructors and Initialization

## Preinit Arrays: `.preinit_array` and `DT_PREINIT_ARRAY`

Before `main()` runs—and even before shared-library constructors execute—the dynamic linker walks **`.preinit_array`** (`SHT_PREINIT_ARRAY`, `DT_PREINIT_ARRAY`). Only the **main executable** may define `.preinit_array` entries; sanitizers like AddressSanitizer (`ASan`) use it to initialize shadow memory before any library constructor allocates on the heap.

## Init Arrays: `.init_array`, `.fini_array` and Their Roles

- **`.init_array` (`SHT_INIT_ARRAY`, `DT_INIT_ARRAY`):** An array of function pointers invoked sequentially **forward** (`[0 .. N-1]`) before `main()`. C++ global object constructors and `__attribute__((constructor))` functions register here.
- **`.fini_array` (`SHT_FINI_ARRAY`, `DT_FINI_ARRAY`):** An array of function pointers invoked in **reverse** (`[N-1 .. 0]`) during normal `exit()` or `dlclose()`.

## Init and Fini Functions: `.init` and `.fini` Sections

Legacy SVR4 binaries concatenated raw machine code fragments into `.init` (`DT_INIT`) and `.fini` (`DT_FINI`) via `crti.o` and `crtn.o`. Modern compilers prefer `.init_array`/`.fini_array` because pointer tables compose cleanly with `--gc-sections`, LTO, and priority sorting without fragile code-fragment splicing.

## Initialization Order: Dependencies, Library Initialization Sequence

`ld.so` initializes shared libraries in **topological dependency order** (dependencies before dependents): if `app` needs `libA` and `libA` needs `libB`, `ld.so` runs `libB` constructors → `libA` constructors → `app` constructors → `main()`. On process exit, destructors run in the exact reverse order.

## `__attribute__((constructor))` and `__attribute__((destructor))`

GCC and Clang accept an optional numeric priority (`101..65535`; `0..100` are reserved for the runtime). Lower numbers run **earlier** for constructors and **later** for destructors:

```c
#include <stdio.h>

__attribute__((constructor(101))) static void init_early(void) {
    puts("1. constructor(101): runs first");
}

__attribute__((constructor(200))) static void init_late(void) {
    puts("2. constructor(200): runs second");
}

__attribute__((destructor(200))) static void fini_first(void) {
    puts("4. destructor(200): runs first on exit");
}

__attribute__((destructor(101))) static void fini_last(void) {
    puts("5. destructor(101): runs last on exit");
}

int main(void) {
    puts("3. inside main()");
    return 0;
}
```

## Hands-On Lab: Verify `.init_array` Priority Ordering

Compile the snippet above and verify both the runtime sequence and the `.init_array` pointer table:

```bash
cc /tmp/ctor_lab.c -o /tmp/ctor_lab && /tmp/ctor_lab
readelf -S /tmp/ctor_lab | grep -E 'init_array|fini_array'
```
