---
title: The Runtime Network Poller (netpoll, epoll, kqueue, IOCP)
---

# The Runtime Network Poller (`netpoll`, `epoll`, `kqueue`, IOCP)

Go's runtime network poller (`src/runtime/netpoll.go`, `netpoll_epoll.go`, `netpoll_kqueue.go`, `netpoll_windows.go`, and `src/internal/poll`) turns non-blocking asynchronous OS socket events into blocking-style synchronous Go code (`conn.Read(buf)`) so 100,000 idle connections consume only ~2 KB stack + ~160 B `pollDesc` each—zero blocked OS threads.

## Bridging Goroutines and Kernel Event Loops

```text
                              Go User Code
                       n, err := conn.Read(buf)
                                  |
                          [1] Non-blocking read
                                  v
                       syscall.Read(fd, buf)
                             /            \
                     [Data Available]   [EAGAIN / EWOULDBLOCK]
                           /                        \
                  Return to User            [2] Register in pollDesc
                                            [3] gopark(waitReasonIOWait)
                                            (Goroutine parked in _Gwaiting)
                                            (Thread M free to run other Gs)
                                                     |
                                                     v
                                          +---------------------+
                                          | OS Kernel Poller    |
                                          | (epoll / kqueue)    |
                                          +----------+----------+
                                                     |
                              [4] epoll_wait detects socket readability
                                                     |
                                                     v
                                          +---------------------+
                                          | runtime.netpoll()   |
                                          | Awakens parked Gs   |
                                          | injectglist()       |
                                          +----------+----------+
                                                     |
                                      [5] G transitions to _Grunnable
                                      [6] Resumes Read(), extracts data
```

## Core Structures: `pollDesc` and `pollCache`

Each network FD attaches to a `pollDesc` allocated from a lock-free 4 KB block pool (`pollCache`). Its atomic `rg` (reader) and `wg` (writer) fields act as state machines (`0 = pdReady`, `1 = pdWait`, or `*g` pointing to the parked goroutine). Deadlines (`SetReadDeadline`) arm runtime timer-heap callbacks (`pd.rt`, `pd.wt`) that wake the parked goroutine to return `os.ErrDeadlineExceeded` without closing the socket.

| Operating System | Engine | Mechanism | Characteristics |
|---|---|---|---|
| Linux | `epoll` | `epoll_create1`, `epoll_ctl` (`EPOLLET`) | Edge-triggered; drains until `EAGAIN` |
| macOS / BSD | `kqueue` | `kqueue()`, `kevent()` (`EV_CLEAR`) | Filter-based (`EVFILT_READ`, `EVFILT_WRITE`) |
| Windows | IOCP | `CreateIoCompletionPort` | Completion-based; Go 1.25+ supports `FILE_FLAG_OVERLAPPED` in `os.NewFile` |

## Scheduler Integration and Go 1.24–1.27 Upgrades

- **Three polling points:** `runtime.findrunnable` (idle work-stealing fallback), every 61st `schedtick` fairness check, and `sysmon` every 10 ms (`injectglist` + `startm`).
- **Linux MPTCP default (Go 1.24+):** `net.ListenConfig` enables Multipath TCP by default on supported Linux kernels.
- **Windows overlapped handles (Go 1.25+):** `os.NewFile` binds `FILE_FLAG_OVERLAPPED` handles directly to IOCP.
- **HTTP/2 RFC 9218 priority & QUIC handshake polling (Go 1.27):** `Server.DisableClientPriority` and `tls.QUICConfig.ClientHelloInfoConn`.

Observe netpoller wakeups live with `GODEBUG=netpoll=1 go run server.go`, and always set explicit `ReadTimeout`, `WriteTimeout`, and `IdleTimeout` so unresponsive peers cannot pin `_Gwaiting` goroutines indefinitely.

## Related Chapters

- [201 Scheduler GMP Deep Dive](201-scheduler-gmp-deep-dive.md)
- [266 epoll Inside the Go Scheduler](266-epoll-inside-the-go-scheduler.md)
- [01 Netpoller and Goroutine Scaling](../14-network-performance/01-netpoller-and-goroutine-scaling.md)
