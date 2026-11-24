---
title: 'Interactive Debugging: Delve, VSCode, GoLand & Docker'
---

# Interactive Debugging: Delve, VSCode, GoLand & Docker

When logs and static reading aren't enough, **Delve (`dlv`)** lets you pause execution, inspect local variables and goroutines, and step through live Go code. Both VSCode and GoLand use Delve under the hood.

## Installing Delve

```bash
go install github.com/go-delve/delve/cmd/dlv@latest
dlv version
```

> [!NOTE]
> Always debug unoptimized builds when possible (`dlv debug` does this automatically). Optimized binaries may inline functions or eliminate variables, making breakpoints and `print` inspections confusing.

## Setting Up the Debugger in VSCode

Install the official **Go (`golang.go`)** extension, then add `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Package",
      "type": "go",
      "request": "launch",
      "mode": "debug",
      "program": "${workspaceFolder}/cmd/api",
      "args": ["--port=8080"],
      "env": {
        "LOG_LEVEL": "debug"
      }
    }
  ]
}
```

- Click the gutter next to a line number to set a breakpoint (right-click for **Conditional Breakpoint** or **Logpoint**).
- Press `F5` to start debugging; use the **Call Stack** and **Variables** panels to inspect values and switch goroutines.

## Setting Up the Debugger in GoLand

GoLand ships with Delve integrated:
1. Open the file containing `func main()` or a `Test*` function.
2. Click the green gutter icon next to `main` or `Test*` and choose **Debug '...'**.
3. Use **Run → Edit Configurations** to add CLI arguments, environment variables, or Go build tags.
4. Right-click any breakpoint to add a condition (e.g., `userID == 42`) or use **Evaluate Expression** (`Alt+F8`) while paused.

## Debugging Nil Pointers, Goroutines, Tests, and HTTP Requests

### Debugging Tests

- **IDE:** Click **Debug Test** directly above any `func TestFoo(t *testing.T)`.
- **CLI:**
  ```bash
  dlv test ./internal/billing -- -test.run TestCalculateTotal
  ```

### Debugging Goroutines

While paused at a breakpoint in Delve CLI:

```text
(dlv) goroutines          # List all goroutines and their states
(dlv) goroutine 14        # Switch context to goroutine 14
(dlv) bt                  # Print backtrace for goroutine 14
(dlv) locals              # Show local variables in the current frame
```

In VSCode and GoLand, the **Call Stack** / **Frames** panel lists every active goroutine so you can click between them directly.

### Debugging an HTTP Request

1. Set a breakpoint inside your `http.HandlerFunc` (e.g., on `r.URL.Query()` or JSON decoding).
2. Start the server under the debugger (`F5` / `dlv debug`).
3. Trigger the endpoint with `curl http://localhost:8080/orders?id=7`.
4. Step over (`F10`) line by line to inspect request headers, decoded structs, and error returns.

## Debugging a Go Application in Docker

To debug a Go service running inside a Docker container, compile with optimizations and inlining disabled (`-gcflags="all=-N -l"`), include Delve in the container, and expose Delve's headless DAP/API port (`2345`):

```dockerfile
FROM golang:1.25 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -gcflags="all=-N -l" -o /server ./cmd/api
RUN go install github.com/go-delve/delve/cmd/dlv@latest

FROM debian:bookworm-slim
COPY --from=builder /server /server
COPY --from=builder /go/bin/dlv /usr/local/bin/dlv
EXPOSE 8080 2345
CMD ["dlv", "--listen=:2345", "--headless=true", "--api-version=2", "--accept-multiclient", "exec", "/server"]
```

Run the container with both ports mapped:

```bash
docker run --rm -p 8080:8080 -p 2345:2345 --security-opt="seccomp=unconfined" --cap-add=SYS_PTRACE my-go-debug
```

Then attach from VSCode (`"request": "attach"`, `"mode": "remote"`, `"port": 2345`, `"substitutePath"` mapping container `/app` to `${workspaceFolder}`) or GoLand (**Go Remote** run configuration on `localhost:2345`).

> [!WARNING]
> Never ship headless Delve or `-gcflags="all=-N -l"` in production images. Keep debug targets in a separate Dockerfile stage.

## Related Chapters

- [09 Debugging by Eye & Reading Panic Traces](09-debugging-by-eye-and-panics.md)
- [238 Production Debugging Tooling](../25-go-deep-dives/238-production-debugging-tooling.md)
- [16 Debugging and Diagnosing Network Problems](../12-socket-programming/16-debugging-and-diagnosing-network-problems.md)
