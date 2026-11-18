# ltrace

## Overview

`ltrace` intercepts and prints **dynamic library calls** made by a process (such as `malloc`, `fopen`, `strcmp`, `getenv`, or OpenSSL functions), similar to how `strace` traces kernel syscalls. It is ideal for seeing what configuration keys, strings, or library functions a binary calls at runtime without attaching a full source-level debugger.

```bash
sudo apt install ltrace   # Debian/Ubuntu
sudo dnf install ltrace   # Fedora/RHEL
```

## Syntax

```bash
ltrace [options] command [args...]
ltrace -p PID
```

## Common Options

| Option | Description |
|--------|-------------|
| `-e filter` | Trace only matching library symbols (e.g., `-e malloc+free`, `-e '*ssl*'`) |
| `-p PID` | Attach to an already-running process |
| `-c` | Count calls, time, and errors, printing a summary table on exit |
| `-T` | Show time spent inside each call |
| `-t` / `-tt` | Prefix each line with wall-clock timestamp (`-tt` adds microseconds) |
| `-f` | Follow child processes created via `fork`/`clone` |
| `-S` | Display system calls in addition to library calls |
| `-C` | Demangle C++ symbol names into human-readable signatures |
| `-o file` | Write trace output to a file instead of stderr |
| `-s N` | Capture up to `N` bytes per string argument (default is 32) |

## Key Use Cases

1. Inspect which environment variables (`getenv`) or config keys a closed-source binary checks
2. Track memory allocation patterns (`malloc`, `calloc`, `realloc`, `free`) or leaks
3. Debug string comparisons (`strcmp`, `strncmp`) or regex/TLS library calls
4. Profile which library calls consume the most time (`-c -T`)

## Safety

- Attaching to a live process pauses threads briefly and can add significant CPU overhead—avoid on latency-sensitive production services.
- Requires `ptrace` permissions (`kernel.yama.ptrace_scope`); attaching to non-child processes typically requires `sudo` or `CAP_SYS_PTRACE`.
- Trace output can contain secrets (passwords passed to `strcmp`, tokens in `getenv`); handle `-o` trace files carefully.

## Examples with Explanations

### Check which environment variables a binary reads

```bash
ltrace -e getenv ./myapp 2>&1 | head -n 30
```

Reveals undocumented feature flags or environment variables checked at startup.

### Trace file-open and string calls with full string arguments

```bash
ltrace -s 256 -e 'fopen+fclose+strcmp' ./myapp --config /etc/app.conf
```

`-s 256` prevents long file paths or comparison strings from being truncated at 32 characters.

### Summarize slowest and most frequent library calls

```bash
ltrace -c -T ls -la /usr
```

Prints a ranked summary table of call counts, total time, and microseconds per call.

### Attach to a running service and follow forks

```bash
sudo ltrace -f -tt -p "$(pidof myapp)" -e 'malloc+free' -o /tmp/ltrace.log
```

Logs timestamped allocations across worker processes to `/tmp/ltrace.log` without cluttering the terminal.

### Demangle C++ symbols

```bash
ltrace -C ./cpp-service
```

Turns mangled names like `_ZNSt7__cxx1112basic_string...` into readable C++ method signatures.

### Combine library calls and syscalls

```bash
ltrace -S -e 'open*+fopen*' curl -s https://example.com > /dev/null
```

`-S` bridges `ltrace` and `strace` so you can see a library wrapper call (`fopen`) and the underlying kernel syscall (`openat`) together.

## Understanding Output

Typical `ltrace` line:

```text
getenv("HOME")                                    = "/home/deploy" <0.000142>
fopen("/home/deploy/.apprc", "r")                 = 0 <0.000089>
```

| Part | Meaning |
|------|---------|
| Function & args | Library symbol and captured arguments (truncated unless `-s` is raised) |
| `= value` | Return value (`0` / `NULL` for failed pointers) |
| `<0.000142>` | Time spent in the call in seconds (shown with `-T`) |

With `-c`, `ltrace` outputs `% time`, `seconds`, `usecs/call`, `calls`, and `function`.

## Notes & Pitfalls

- **Statically linked binaries** (including many Go binaries and Alpine/musl static builds) do not use dynamic PLT calls, so `ltrace` will show little or nothing—use `strace` or `perf` instead.
- **Rust and stripped C++ binaries** may have inlined calls or fewer PLT symbols; `ltrace` only hooks calls crossing dynamic library boundaries.
- Modern distros with **CET / IBT or hardened PLT/GOT** or `ptrace_scope=1` may block non-root attaching; run as root or launch the binary directly under `ltrace`.
- For kernel/file/network issues (`ENOENT`, `EACCES`, `ECONNREFUSED`), `strace` is usually faster and clearer than `ltrace`.

## Related Commands

- `strace` — trace kernel system calls
- `perf` — low-overhead CPU and event profiling
- `ldd` / `nm` — inspect shared library dependencies and exported symbols
- `gdb` — interactive source/assembly debugger

## Additional Resources

- `man ltrace`
- `man strace`
