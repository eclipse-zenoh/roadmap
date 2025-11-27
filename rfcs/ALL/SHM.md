# Zenoh SHM

**Supported platforms:**
- Linux
- MacOS (and other BSD)
- Windows

**API support:**
- Rust
- C
- C++
- Python

## General Concepts

Zenoh Shared Memory (SHM) provides zero-copy data transfer between processes on the same host while preserving the standard `ZBytes` interface. The core buffer types are `ZShm` (immutable) and `ZShmMut` (mutable), which can be wrapped in `ZBytes` for compatibility with existing code. In practice, a publisher allocates a `ZShmMut`, wraps it in `ZBytes`, and sends it. A subscriber on the same host can receive the buffer without copying (it still points to the original shared memory). If a subscriber does not support SHM (or is on a different host), Zenoh automatically converts the shared buffer into a normal in-memory copy at the boundary of the SHM domain. In that case, the subscriber receives a normal copy of the data over the network.

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

**Fewer Constraints**

- Buffer lifetime: An SHM buffer can live indefinitely and is not tied to any single session or process.
- Shallow copies: You can take unlimited references (shallow copies) to a shared buffer.
- Multiple publications: The same buffer can be published multiple times, even through different Zenoh sessions.
- Network hops: There is no restriction on hops or network topology within the same host.
- Anywhere `ZBytes` is accepted: SHM buffers are fully compatible with any Zenoh API that consumes `ZBytes`.

**Interoperability**

Zenoh sessions automatically negotiate SHM support when doing Zenoh handshake. During the negotiation, each link partner node claims SHM support, SHM version and a list of SHM protocols that partner *can read*. As part of this negotiation, link partner nodes also make mutual SHM availability check - making sure that they can access each other's SHM segments.

If sender sends an SHM buffer but a receiver (or an intermediate node) does not support shared memory by negotiation — Zenoh transparently falls back to copying. In that case, the buffer is converted into a regular `ZBytes` payload at the edge of the SHM domain, and the subscriber receives the data normally over the network.

**Robustness**

Zenoh SHM is built with robustness as a primary design goal, providing predictable and safe behavior even under heavy load, high concurrency, or constrained system resources.

Key robustness properties

1. Decentralized ownership model
Each Provider independently manages its own SHM segments.
This design removes global contention, eliminates single points of failure, and isolates faults—ensuring that memory corruption within one Provider cannot impact others.

2. Automatic garbage collection of lost buffer references
Zenoh includes built-in mechanisms to detect and reclaim buffers whose references have been lost, preventing memory leaks over long runtimes.
Reference loss may occur in scenarios such as:

      - A process that holds buffer references crashes
      - Transport losses when using an unreliable protocol (e.g., UDP)
      - Network reconnects due to interruptions or topology changes

Currently, the worst-case lost buffer reference reclamation delay is 100ms.

For mission-critical deployments, the Zenoh team can provide extended-support guidelines on designing SHM usage patterns that contain and recover from corruption (e.g., due to undefined behavior or unsafe writes).

> **Best practices**
> 
> - Because reclaiming lost SHM buffers can incur additional latency, design systems to minimize lost buffer references. In particular, prefer the Reliable reliability option for SHM publications and use it together with reliable transport protocols.
> > Note: this recommendation does not apply to congestion-control `Drop` behavior — messages intentionally dropped by congestion control are not treated the same as lost references.
> 
> - Avoid `Block` under heavy congestion. Sending SHM messages over transports that are heavily congested and use the congestion-control mode `Block` can cause messages to be invalidated in-flight. Under severe congestion, `Block` can cause publishers to stall and may lead to messages being garbage-collected to free resources, producing unexpected message loss.

**Buffer memory management**

Each `ZShm` buffer is reference-counted and Zenoh maintains metadata to handle dangling references (for example, if a process crashes while holding a buffer).

Buffers are allocated by `SharedMemoryProvider`. Providers are extensible and capable of using pluggable backends which implement an allocator and utilize some SHM system API. The only backend currently shipped with Zenoh by default is `PosixShmProviderBackend`, which implements a general-purpose allocator and uses POSIX shared memory.

Allocated buffers are garbage collected when all the corresponding references across zenoh network are dropped. The garbage collection process is currently triggered and run in the context of following operations:
- SHM buffer allocation with any type of allocation policy that intends GC'ing.
- `garbage_collect()` method of `SharedMemoryProvider`

> **Best practices**
> 
> - The optimal `SharedMemoryProvider` capacity is usually something around twice the sum of in-flight payloads. Providing too small amount of memory might cause out-of-memory hickups while providing too big memory regions is not cache-friendly.
> 
> - GC'ing might take time and produce latency jitter, especially if there are many buffers in-flight. If this becomes an issue, consider optimizing time points when you allocate and\or use separate `garbage_collect()` method of `SharedMemoryProvider`.

## SHM Providers

Zenoh SHM allocator implements a two-layer model: an `ShmProvider` (the object the rest of Zenoh talks to) and an `ShmProviderBackend` (the concrete implementation that knows how to allocate and manage the shared memory — POSIX, custom kernel API, special allocation strategies, etc.). This makes it straightforward to plug your own backend if you need a custom allocator or a different OS/shared-memory API.

**ShmProvider**: the high-level interface Zenoh code (sessions, publishers, subscribers) uses to allocate, reference and publish SHM buffers. It provides allocation policies (block-on, defragment, garbage-collect, etc.), alignment control, manual invalidation, and reference-counted buffer handles that survive abnormal process termination robustly. 

**ShmProviderBackend**: the low-level “pluggable” implementation — it encapsulates the platform/strategy (POSIX shm_open/mmap, System V, a kernel driver, a pre-allocated hugepage region, or a vendor-specific API). The backend exposes the primitive operations (allocate chunk, free chunk, defragment) and a protocol ID so different backends/clients can be kept distinct.

Use cases for a custom backend:

- Use a non-POSIX SHM mechanism (e.g., vendor kernel module, DPDK hugepage pool, specialized RTOS API).

- Implement custom allocation strategies.

- Interpose custom safety / security checks on buffer creation.

- Use hardware NIC-attached memory, or a memory region shared with other subsystems.

## Allocation policies

To give users fine-grained control over memory allocation and garbage-collection behavior, the Zenoh SHM API provides allocation policies. Each allocation request can be parameterized with a specific policy that defines how the allocator should behave.
In the Rust API, multiple allocation policies can be chained to create more sophisticated allocation strategies.

Available policies

- `JustAlloc` — Attempts to allocate memory once and returns an error on failure. This fails immediately if no free memory is available in the provider.

- `GarbageCollect` — If the allocation cannot proceed due to insufficient memory, this policy triggers garbage collection and retries the allocation.

- `Defragment` — If allocation fails due to memory fragmentation, this policy attempts to defragment memory before retrying.

- `BlockOn` — If allocation fails, this policy blocks the caller until the allocation can succeed.

For usage examples, see the [z_alloc_shm](https://github.com/eclipse-zenoh/zenoh/blob/main/examples/examples/z_alloc_shm.rs)

## Implicit transport optimization

Zenoh SHM performs an automatic transport optimization to minimize copies for large payloads. If session sends a non-SHM `ZBytes` payload that exceeds a configurable size threshold and the link partner node supports SHM, Zenoh will implicitly convert that payload into an SHM buffer (copying the data once into a `ZShm` region). Currently, this conversion happens separately for each link partner node.

Each Zenoh Session uses own internal `ShmProvider` for this type of optimization. The usage policy of this provider is lazy-opportunistic: provider is created lazily in separate blocking task on the first demand and used only if available - only when initialization task completed and provider has free memory to perform allocation. In other words, it is not guaranteed that optimization will happen for every buffer.

This behavior is especially valuable for routers or gateways that receive large packets from the network and must forward them to several local processes.

The implicit transport optimization settings can be found in our [config JSON](https://github.com/eclipse-zenoh/zenoh/blob/3b0cef3049bdad06b113af4156477560cae327ee/DEFAULT_CONFIG.json5#L737).

## Typed API (Rust only)

Besides common raw-byte-oriented API (`ZShm` and `ZShmMut`) typed API is also supported through `Typed` and `TypedLayout` generics. See [Allocation API](https://github.com/eclipse-zenoh/zenoh/blob/main/examples/examples/z_alloc_shm.rs) and [zshm](https://github.com/kydos/zshm) examples.

## On the wire

From the wire perspective, an SHM buffer reference is transmitted as a small Zenoh message with a 16-byte payload. It travels over the same transport links and configurations as any other Zenoh message, making SHM fully compatible with existing link setups.

## Virtual memory management

Zenoh pre-commits and locks shared-memory pages. This means once a buffer is allocated, it will not fault or be swapped out during use, ensuring consistent performance.

> **Best practices**
> 
> - Make sure your system has enough free RAM to hold all SHM segments, since our memory is not overcommitted. 
> 
> - On Linux, raise the memlock limit if needed (for example, `ulimit -l unlimited`) so Zenoh can lock the necessary memory.

## Docker

Zenoh SHM is fully compatible with Docker.

> **Best practices**
> 
> - Ensure relevant containers share `/dev/shm` or a named volume for shared memory.
> 
> - On macOS/BSD Docker, also share the host’s `/tmp` (Rust’s default `temp_dir`) across containers.

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

> **Best practices**
> 
> - Always perform an orderly shutdown of your Zenoh resources on process termination: close sessions/transports, drop buffers, stop the SHM provider, then exit normally. This makes the destructor-based GC work reliably.
> 
> - Avoid using low-level exit paths that skip normal teardown (e.g., `_exit()`), and handle signals that may interrupt normal shutdown. Register graceful shutdown handlers so cleanup runs even on `SIGINT`/`SIGTERM`.
> 
> - If you expect processes to crash or be killed frequently, rely on the dangling GC as a safety net — but still try to minimize crashes.
> 
> - Provide an explicit operator/maintenance path in your deployment to trigger the user API cleanup for special cases (for example, during maintenance or automated restarts).

## Compilation

In order to be able to receive and retransmit SHM buffers, you need to compile with `shared-memory` feature. In order to create new SHM buffers, you need to have both `shared-memory` and `unstable` features because SHM API is unstable.

## Examples

- Allocation API: https://github.com/eclipse-zenoh/zenoh/blob/main/examples/examples/z_alloc_shm.rs
- SHM Buffer API: https://github.com/eclipse-zenoh/zenoh/blob/main/examples/examples/z_bytes_shm.rs
- How to publish SHM Buffer: https://github.com/eclipse-zenoh/zenoh/blob/main/examples/examples/z_pub_shm.rs
