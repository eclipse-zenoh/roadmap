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

## Compilation

In order to be able to receive and retransmit SHM buffers, you need to compile with `shared-memory` feature. In order to create new SHM buffers, you need to have both `shared-memory` and `unstable` features.

## Examples

- Allocation API: https://github.com/eclipse-zenoh/zenoh/blob/main/examples/examples/z_alloc_shm.rs
- SHM Buffer API: https://github.com/eclipse-zenoh/zenoh/blob/main/examples/examples/z_bytes_shm.rs
- How to publish SHM Buffer: https://github.com/eclipse-zenoh/zenoh/blob/main/examples/examples/z_pub_shm.rs
