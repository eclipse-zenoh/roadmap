# The Key Expressions Language

## Preface

Zenoh being a Named-Data oriented protocol, its address space is the space of names of given to data.

Specifically, addressable data in Zenoh is addressed via a *key*.

To provide a framework for data hierarchy, Zenoh defines a key as `/`-separated list of chunks, where each chunk is a non-empty[^1] UTF-8[^2] string that can't contain the following characters: `*$?#`.

However, Zenoh's APIs rely on *Key Expressions* (KE) rather than *keys* to address their operations. This document specifies the Key Expression Language: a small language to express sets of keys.

[^1]: Note that this forbids leading and trailing `/`s, as well as forbids `//` from existing within a key.

[^2]: If you work with strings that contain glyphs that may have several UTF representations, such as accented characters, we suggest you get acquainted with the subject of [Unicode Equivalence](https://en.wikipedia.org/wiki/Unicode_equivalence) to avoid some nasty bugs... Or stick with ASCII if you'd rather stay sane.

## The basics

Key Expressions are a superset of Keys. They allow you to succinctly write sets of keys, and are treated in this document as equivalent to the set they represent.

Thus, two Key Expressions **A** and **B**, just like sets, can be related like so:
* **A** and **B** are disjoint there doesn't exist a key that belongs to both.
* If at least one key belongs to both **A** and **B**, they *intersect*.  
  Intersection is a commutative relationship: (**A** ∩ **B**) = (**B** ∩ **A**).  
  This relation is at the core of Zenoh's operations, as it is the one that determines whether a set of values addressed by **A** is concerned by an operation on **B**.
* If all of **B**'s keys are also contained in **A**, **A** *includes* **B**.  
  This is *almost* an anti-commutative relationship, as unless **A** = **B**, **A** includes **B** implies that **B** *does not* include **A**.  
  Note that **A** includes **B** *always* implies that **A** intersects **B**.
* If **A** includes **B** *and* **B** includes **A**, they are *equal*.

In general, a KE is much like a Key, where any amount of chunks may be replaced by either a wild chunk, or a DSL chunk.

The KE language is designed to ensure *unicity*: there exists *at most* 1 string that is a valid KE for any given set of keys. This implies that string equality and set equality are equivalent when these strings are valid key expressions. This property is achieved through the existence of [canon forms](#canon-forms)

## Wild chunks

Wild chunks are the first and fastest mechanism to write KEs that address multiple keys (an infinite amount, in fact).

They are inspired by globs, and looking at a few examples should give you a good intuition of their behaviours.

Like their names suggest, wild chunks are specific chunks, and not patterns that may appear inside a chunk: they may be preceded and followed *only* by `/`.

### `*`: single wild
`*` expresses "exactly one chunk of any value", and is equivalent to the `[^/]*` regular expression.  

For example, `a/*/b`...
* Includes:
	* `a/c/b`
	* `a/hi/b`
* Intersects:
	* `*/a/b`
	* `*/*/*`
* Is disjoint with:
	* `a/*/c`
	* `b/*/a`
	* `a/hi/there/b`
	* `a/hi/*/b`

### `**`: double wild
`**` expresses "any amount (including 0) of chunks of any values".

`a/**/b`...
* Includes:
	* `a/b`
	* `a/**/b/b`
	* `a/*/b`
	* `a/*/*/b`
	* `a/*/**/b`
	* `a/**/c/**/b`
* Intersects:
	* `**/b`
	* `a/**`
* Is disjoint with:
	* `a/**/b/c`

Note that in a Zenoh network, publishing a `DELETE` on `**` is equivalent to performing a `rm -rf /` on linux, since `**` includes every possible key.

## Extensibility: DSLs
To allow finer grained sets to be defined, Zenoh provides Domain Specific Languages.

DSLs sections start with `$`, and may be contained anywhere *within* a chunk. They follow the `$<DSL_ID><DSL_CONTENT>` pattern, and each DSL is at liberty of capturing as many characters as suits them. 

For now, only a single DSL is supported:

### `$*`: the sub-chunk equivalent of `*`
`$*` behaves just like `*`, as an equivalent to the `[^/]*` regular expression. However, unlike `*`, `$*` may be preceded and/or followed by any character.

Historical users who were accustomed to `*` being able to do exactly that may be surprised, so here's why `$*` was split from `*`'s new restricted role:
* A `*a` chunk used to be *much* slower than `*/a`. By forcing users to write the less appealing `$*a` instead, we hope to discourage them from using these sub-chunk wilds; and to encourage them to build better key spaces instead.
* Just supporting `$*` as part of `*`'s old behaviour made `*` less performant overall, even when used alone in a chunk. By inconviencing non-compliant users, we offer performance benefits to the compliant ones.

`a/c$*/b`...
* Includes:
	* `a/cool/b`
* Intersects:
	* `a/*/b`
	* `a/$*c/b`
* Is disjoint with:
	* `a/uncool/b`

### Future plans
The Zenoh team plans to introduce more complex DSLs in the future. However, these plans are too blury yet to be made public, and are subject to maintaining the current properties of key expressions.

### Note

KEs containing DSLs WILL put more strain on your Zenoh infrastructure than KEs that don't, so we advise designing your key space such that they don't become necessary. For example, avoid `factory-12/room-10/robot-9`, favoring `factory/12/room/10/robot/9`, as `factory/12/room/10/robot/*` will put less strain on your infrastructure than `factory-12/room-10/robot-$*`.

## Canon forms
Wilds and DSLs by themselves introduce the possibility of multiple strings defining the same sets.

To ensure the very useful *unicity* property, key expressions have a canon form. While more efficient algorithms exist to canonize a string, we will express it as a replacement loop for simplicity. As such, a string is canonized when none of the following operations can be applied any more:
* Any contiguous sequence of `$*`s is replaced by a single `$*`.
* Any contiguous sequence of `**` chunks is replaced by a single `**` chunk.
* Any `$*` chunk is replaced by a `*` chunk.
* `**/*` is replaced by `*/**`.

Non canon forms are forbidden on a Zenoh network, and the expected behaviour if one makes it onto the network is for the next router to drop the message associated with it.

Zenoh's APIs are designed so that the Key Expressions your provide are validated, returning errors if they were not valid. To ensure your KE string is only validated once, construct a `KeyExpr` from it, and use that `KeyExpr` whenever possible. If you're certain that your string is a valid KE, you may use unsafe constructors to bypass these checks.
