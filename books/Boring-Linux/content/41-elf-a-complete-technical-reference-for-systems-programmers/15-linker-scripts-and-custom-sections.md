# Chapter 15: Linker Scripts and Custom Sections

## Linker Script Language: `INPUT`, `SECTIONS`, `OUTPUT_FORMAT`

Every GNU `ld` or LLVM `lld` invocation is driven by a **linker script** (view the built-in default with `ld --verbose`). The script dictates how input sections from `.o` files map into output sections, virtual addresses, alignment boundaries, and program segments:

- `OUTPUT_FORMAT("elf64-x86-64")` / `ENTRY(_start)`: Declares target format and entry symbol.
- `INPUT(a.o b.o)`: Adds object files implicitly.
- `MEMORY { ... }`: Declares physical Flash/SRAM regions (critical for kernels and bare-metal firmware).
- `SECTIONS { ... }`: Maps input patterns like `*(.text .text.*)` into output sections using the location counter `.`.

## Section Assignment and Memory Region Control

```ld
ENTRY(_start)

MEMORY {
  FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
  SRAM  (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS {
  .text : {
    *(.isr_vector)
    *(.text .text.*)
    *(.rodata .rodata.*)
  } > FLASH

  .data : {
    _sdata = .;
    *(.data .data.*)
    _edata = .;
  } > SRAM AT > FLASH   /* VMA in SRAM, LMA in FLASH */

  .bss (NOLOAD) : {
    _sbss = .;
    *(.bss .bss.* COMMON)
    _ebss = .;
  } > SRAM
}
```

## Custom Sections: Creating and Populating `.my_section`

Linux kernel modules (`__init`, `EXPORT_SYMBOL`), unit-test registries, and CLI plugin tables all use custom ELF sections. When a custom section name is a valid C identifier (no dots, e.g., `my_plugins`), `ld` automatically synthesizes `__start_my_plugins` and `__stop_my_plugins` symbols!

## Linker Script Variables and Symbols

Linker symbols like `_sbss` have **no storage**—only an address. In C, always declare them as `extern char _sbss[];` and take their address (`&_sbss` or `(uintptr_t)_sbss`), never dereference them as an integer value.

## Advanced Layouts: Discontiguous Sections, Padding, Alignment

Use `. = ALIGN(4096);` to isolate read-only and writable sections onto distinct MMU pages, `FILL(0x90909090)` to pad code gaps with `NOP`s, and `KEEP(*(.my_plugins))` so `--gc-sections` does not discard unreferenced plugin descriptors.

## Example: Building a Custom Layout with GCC and LD Scripts

```c
#include <stdio.h>

typedef void (*plugin_fn)(void);
#define REGISTER_PLUGIN(fn)     static plugin_fn const __ptr_##fn     __attribute__((used, section("my_plugins"))) = fn

static void plug_a(void) { puts("plugin A executed"); }
static void plug_b(void) { puts("plugin B executed"); }
REGISTER_PLUGIN(plug_a);
REGISTER_PLUGIN(plug_b);

extern plugin_fn __start_my_plugins[];
extern plugin_fn __stop_my_plugins[];

int main(void) {
    for (plugin_fn *p = __start_my_plugins; p < __stop_my_plugins; p++)
        (*p)();
    return 0;
}
```

Compile and run (`cc /tmp/plug.c -o /tmp/plug && /tmp/plug`), then inspect `readelf -S /tmp/plug | grep my_plugins`.
