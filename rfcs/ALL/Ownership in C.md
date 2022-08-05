# Reflecting ownership through the type system in C
## Preface
There are two ways to use Zenoh from C:
- [zenoh-c](https://github.com/eclipse-zenoh/zenoh-c), the C bindings for [zenoh-rs](https://github.com/eclipse-zenoh/zenoh), the reference implementation for Zenoh.
- [zenoh-pico](https://github.com/eclipse-zenoh/zenoh-pico), a native C library which implements a subset of Zenoh's functionalities, with a focus on embedded devices.

Both of these now share a common API for the most part, so this RFC applies to both.

Zenoh leverages C's type system to help C developpers with the always complicated issue of memory management, by creating a distinction between owned and loaned (or aliasing) types. In doing so, Zenoh aims to help you avoid memory-leaks more easily, and understand for how long you can use your values by reading function signatures alone.

## The core problem, swimming without a POD
Not all types are POD (Plain Old Data) that one can copy around haphazardly. In fact, when working with networks and callbacks, these well-behaved types tend to make themselves scarce in the favour of types that represent allocated ressources, which must be deallocated at some point to avoid memory leaks, unclosed files, etc... For the rest of this RFC, we'll refer to allocating any type of ressource as "construction", and deallocating it as "destruction" or "dropping".

But when to destroy a ressource? How can you make sure you're not pulling the rug from under some other task?

## Owned types and move semantics
One solution would be to copy all the things. If some function needs a value for the long-term, let it copy that value. That way, any function can happily call any value it's created's destructor. But that's wasteful.

In come *move semantics*: when a function needs to a value for the long-term, it may take that value from its caller, assuming the responsibility for destroying it in the caller's stead (this doesn't mean destruction will happen by the end of that specific function, but that the caller is no longer the one responsible for destroying it).

Here's how this translates in the Zenoh API:
- Types that require destruction are marked by the `z_owned` prefix: you should never copy these types, as they aren't POD.
- All of these owned types have some form of *tombstone* state, which lets the destructor detect double-free attempts, and lets you `z_check` whether they are currently in that tombstone state, or if they still have to be dropped.
- Any function that takes a mutable pointer to a `z_owned`-typed value takes responsibility for destroying it. By the time that function returns, the value to which you passed that mutable pointer MUST have been replaced with the tombstone (if you happen to find a function that fails to do so, this is considered a bug, please file an issue). To help you see these moves better, we suggest you use the `z_move` macro, which simply resolves to `&`, but is easier to grep for.

## Loan types: because who wants to only use a value once?

## Some examples
