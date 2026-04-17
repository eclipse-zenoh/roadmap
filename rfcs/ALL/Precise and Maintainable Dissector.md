# Towards a Precise and Maintainable Dissector

- Date: April 2026
- Tracking issue: TBD

## Background

[eclipse-zenoh/zenoh-dissector] was originally written in pure Lua by @cguimaraes (October 2022)—see
the ['The Blue Dragon meets the Wire’s Shark'] blog post—and it would remain as such up until the
release of Zenoh 1.0 (September 2023) where @YuanYuYuan rewrites the codebase in commit
[06c07d5](https://github.com/eclipse-zenoh/zenoh-dissector/commit/06c07d5cde1cfe2c792b7c5382353d5efe69b860)
to leverage Rust, [zenoh-codec] and the [epan-sys] libwireshark bindings.

This new architecture makes it trivial to follow changes in Zenoh's wire encoding of messages, which
essentially reduces to supporting new extensions. The new dissector also achieves better performance
by virtue of being written in a native programming language and by benefiting from upstream
optimizations (e.g. [eclipse-zenoh/zenoh#2378]).

There were two downsides to this new architecture. First, there are no community-maintained
libwireshark Rust bindings. Thus the burden of supporting epan-sys builds—including building
Wireshark from source if not installed on the host—on Linux, macOS and Windows for each new
Wireshark release falls on the shoulders of the Eclipse Zenoh maintainers.

Second, zenoh-codec does not expose the sufficient information needed to build full field-to-byte
mappings between protocol fields and packet payload ranges. This means that zenoh-dissector does not
properly highlight packet bytes for the various Zenoh protocol tree fields and branches.

## Status quo

In April 2025 user @Hugal31 would announce a new Zenoh dissector: [Hugal31/zenoh-c-dissector] whith
the goal of supporting field-to-byte mapping, key-expression resolution and reply-to-query mapping.
@Hugal31 would explain that zenoh-dissector's reliance on zenoh-codec meant that field-to-byte
mapping was not possible, and that the epan-sys interface made it more difficult to access Wireshark
API.

As the name suggests, zenoh-c-dissector is written in both C and Lua. Many dissection routines are
implemented in C in addition to uncompression support which utilizes liblz4. zenoh-c-dissector also
includes support for the Zenoh scouting protocol.

A year later, in April 2026, @kydos would build the third Zenoh dissector, [kydos/zenoh-wireshark].
Similarly to zenoh-c-dissector, zenoh-wireshark bypasses zenoh-codec to implement full field-to-byte
mapping. zenoh-wireshark also circles back to zenoh-dissector's original architecture and opts for a
full Lua codebase. The result is a slower, more difficult to maintain but more feature rich and
easier to install dissector.

| Project | Wireshark API | Decoding strategy | Feature set |
| --- | --- | --- | --- |
| eclipse-zenoh/zenoh-dissector | epan-sys | zenoh-codec | Heuristic dissection (transport), Source/Dest ZID fields |
| Hugal31/zenoh-c-dissector | Lua API | Custom (C) | Field-to-byte mapping, heuristic dissection (scouting), scouting protocol, Key-expression resolution, reply-to-query mapping, uncompression |
| kydos/zenoh-wireshark | Lua API | Custom (Lua) | Field-to-byte mapping, heuristic dissection (transport), scouting protocol, key-expression resolution |

Note that the three dissectors define different protocol fields, thus filters are not interoperable
between them.

## Potential for consolidation

Consequently, the present RFC proposes a new zenoh-dissector architecture with the goal of
preventing further ecosystem fragmentation. This new architecture would have the following three
broad strokes:

1. Drop the dependency on epan-sys, use the Wireshark Lua API instead.
2. Rethink the dependency with zenoh-codec to support field-to-byte mapping: either (A) use lower-level
   zenoh-codec APIs or (B) augment existing high-level APIs to extract needed information.
3. Maintain current performance characteristics by opportunistically offloading slow (decoding)
   routines across the Lua-Rust FFI boundary similarly to zenoh-c-dissector's approach.

It's not yet clear if such an architecture is feasible, thus the following action points are needed:

1. Benchmark the three existing dissectors to better understand the performance implications of each
   architecture.
2. If (1) indicate an acceptable performance regression in a hybrid Rust/Lua codebase, design a new
   interface between zenoh-dissector and zenoh-codec in order to strike the right balance between
   performance, maintainability and field-to-byte mapping support.

If accepted, this RFC would be implemented through a meta tracking issue for action points (1) and
(2). A new zenoh-dissector architecture would be discussed by the community (with the various
dissector maintainers in particular) on [eclipse-zenoh/roadmap] issue threads.

Issues with filter name standardization are out of the scope of this RFC.

[eclipse-zenoh/zenoh-dissector]: <https://github.com/eclipse-zenoh/zenoh-dissector>
['The Blue Dragon meets the Wire’s Shark']: <https://zenoh.io/blog/2023-01-17-zenoh-wireshark/>
[zenoh-codec]: <https://crates.io/crates/zenoh-codec>
[epan-sys]: <https://github.com/eclipse-zenoh/zenoh-dissector/tree/main/epan-sys>
[eclipse-zenoh/zenoh#2378]: <https://github.com/eclipse-zenoh/zenoh/pull/2378>
[Hugal31/zenoh-c-dissector]: <https://github.com/Hugal31/zenoh-c-dissector>
[kydos/zenoh-wireshark]: <https://github.com/kydos/zenoh-wireshark>
[eclipse-zenoh/roadmap]: <https://github.com/eclipse-zenoh/roadmap>
