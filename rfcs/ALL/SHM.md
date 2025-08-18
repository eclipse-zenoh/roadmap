# Zenoh SHM

**Supported platforms:**
- Linux
- MacOS (and other BSD)
- Windows

**API support:**
- Rust
- C
- C++

## General Concepts

Zenoh SHM is designed to provide true zero-copy optimization abstracted behind the classic ZBytes API. The basic SHM data buffer type is `ZShm` (or its mutable representation `ZShmMut`) which is typically wrapped into ZBytes, providing smooth integration into generalized application code:
```
________________________________                       ____________________________________
|                              |                       |                                  |
|        SHM Publisher         |     _____________     |     Non-SHM-aware Subscriber     |
|                              |     |           |     |                                  |
| alloc() -> ZShmMut -> ZBytes-|---->| Localhost |-----|-> ZBytes (implicitly wraps ZShm) |
|______________________________|     |   Zenoh   |     |__________________________________|
                                     |  Network  |                   
                                     |___________|     ______________________________________
                                           |           |                                    |
                                           |           |        SHM-aware Subscriber        |
                                           |           |                                    |
                                           |-----------|--> ZBytes (implicitly wraps ZShm)  |
                                                       | convertable to ZShm and/or ZShmMut |
                                                       |____________________________________|
```

There are no limits on SHM buffer lifetime, number of shallow copies, number of republications, number of hops, or network topology. SHM buffers are not pinned to a particular Zenoh Session - it is possible to publish the same buffer multiple times through different Sessions. SHM buffers can be used anywhere ZBytes is used.

Zenoh Sessions probe and negotiate SHM support. For participants not supporting SHM (due to their configuration, compilation flags, access rights, or non-localhost location), any published SHM buffer will be implicitly converted to a non-SHM buffer at the last hop before leaving the SHM Domain boundary.

SHM buffers are reference counted with additional mechanics applied to support dangling reference recovery (in case a process holding an SHM buffer crashes) to keep the system robust.

SHM buffers are allocated by `SharedMemoryProvider` objects. Providers are extensible and capable of using pluggable backends which implement an allocator and utilize some SHM system API. The only backend currently shipped with Zenoh by default is `PosixShmProviderBackend`, which implements a general-purpose allocator and uses POSIX shared memory.

## Docker

General Docker shared-memory configuration instructions should be applied to enable container-to-container and/or container-to-host SHM operation.
Additionally, on BSD systems, Docker should also share the tmp directory corresponding to Rust's `std::env::temp_dir()`.

## Memory isolation and corruption safety

Zenoh SHM is fully decentralized and each Provider owns its own set of SHM segments. The Zenoh team provides recommendations on building corruption-safe SHM applications on an extended support basis.

## Segment garbage collection

Zenoh SHM uses POSIX named shared memory as the default backend. POSIX semantics for named SHM objects (created with `shm_open`) require an explicit `shm_unlink` to remove the name — the kernel will not automatically unlink the name when processes exit. Because of that, Zenoh implements two complementary cleanup mechanisms to avoid leaving orphaned / dangling shared-memory segments around.

**POSIX background (short)**

Named POSIX shared-memory objects persist until they are explicitly unlinked (for example with `shm_unlink`). On Linux these objects are visible under `/dev/shm`, so orphaned segments are often observable there. Abnormal process termination or use of low-level exit paths (e.g. `_exit()` or a hard crash) can prevent normal object destructors and language-level cleanup from running, so relying solely on automatic destructor behaviour is unsafe.

**Zenoh mechanics**

***Destructor-based GC***

When every segment owner across all processes is dropped (for example: session transports closed, SHM provider stopped, and all buffer references released), the last process that used the segment unlinks it.
Practical note: orderly shutdown is required for this to work reliably. Abnormal termination or using APIs that bypass normal teardown can prevent destructors/cleanup from running.

***Dangling-segment GC***

To handle cases where an owner crashed or called an exit path that didn’t run teardown, Zenoh includes a dangling-segment detection and cleanup routine. This routine can detect leftover segments and remove them; it is triggered in several situations:

1. When a newly started process first attempts to use SHM (Zenoh inspects existing segments and cleans up detected dangling ones).

2. On orderly exit of a SHM-enabled process (additional sanity checks/cleanup can run).

3. When explicitly invoked by the user through the Zenoh API (manual/forced cleanup).

**Recommendations & best practices**

Always perform an orderly shutdown of your Zenoh resources on process termination: close sessions/transports, drop buffers, stop the SHM provider, then exit normally. This makes the destructor-based GC work reliably.

Avoid using low-level exit paths that skip normal teardown (e.g., `_exit()`), and handle signals that may interrupt normal shutdown. Register graceful shutdown handlers so cleanup runs even on `SIGINT`/`SIGTERM`.

If you expect processes to crash or be killed frequently, rely on the dangling GC as a safety net — but still try to minimize crashes.

Provide an explicit operator/maintenance path in your deployment to trigger the user API cleanup for special cases (for example, during maintenance or automated restarts).

## Compilation

In order to be able to receive and retransmit SHM buffers, you need to compile with `shared-memory` feature. In order to create new SHM buffers, you need to have both `shared-memory` and `unstable` features.

## Examples

- Allocation API: https://github.com/eclipse-zenoh/zenoh/blob/main/examples/examples/z_alloc_shm.rs
- SHM Buffer API: https://github.com/eclipse-zenoh/zenoh/blob/main/examples/examples/z_bytes_shm.rs
- How to publish SHM Buffer: https://github.com/eclipse-zenoh/zenoh/blob/main/examples/examples/z_pub_shm.rs
