# Selectors
## Preface
With [Key Expressions](./Key%20Expressions.md), Zenoh provides a way to act on a lot of data through a single operation.

The original goal of selectors was to allow selecting data not just through their keys, but also through their value (when the data is stored in a self-describing manner, or when the queryable is able to interpret the data).

However, its main use currently is to pass arguments to the queryables when performing requests to them. This can be used notably a mean to pass RPC arguments

## Syntax
The selector syntax is derived from the URL syntax, so let's start with a URL example (with some spacing for legibility):
```
https://some.hostname.com / path/to/something ? arg1=val1&arg2=value%202
^                       ^   ^               ^   ^                      ^
|- Protocol + Hostname -| / |----- Path ----| ? |-- query parameters --|
```
and an similar Zenoh Selector:
```
path/**/something ? arg1=val1&arg2=value%202
^               ^   ^                      ^
|Key Expression-| ? |--- value selector ---|
```

As you can see, Zenoh selectors are basically URLs without protocol and hostname, since the protocol is Zenoh, and the hostname is irrelevant in a Named Data Network.

In fact, this is is leveraged by Zenoh's REST API, where HTTP request to `https://my.zenoh.router:8000 / <Selector>` get turned into the equivalent Zenoh operation on that Selector, if your zenoh router has a [rest plugin](https://github.com/eclipse-zenoh/zenoh/tree/master/plugins/zenoh-plugin-rest) enabled and listening on port 8000.

## The value selector
The value selector functions just like query parameters:
* It's separated from the path (Key Expr) by a `?`.
* It's a `?` list of key-value pairs.
* The first `=` in a key-value pair separates the key from the value.
* If no `=` is found, the value is an empty string: `hello=there&kenobi` is interpreted as `{"hello": "there", "kenobi": ""}`.
* The selector is assumed to be url-encoded: any character can be escaped using `%<charCode>`.

There are however some additional conventions:
* Duplicate keys are considered Undefined Behaviour, but the tools we provide for selector interpretation will always check for duplicates of the interpreted keys, returning errors when they arise.
* The Zenoh Team considers any key that does not start with an ASCII alphabetic character reserved.
* Since Zenoh operations may be distributed over diverse networks, we encourage queryable developpers to use some prefix in their custom keys to avoid collisions.
* When interpreting a key-value pair as a boolean, the absence of the key-value pair, or the value being `false` are the only "falsey" values: in the previous examples, the `kenobi` boolean is considered truthy.

## Reserved Keys
### The Zenoh Namespace
The Zenoh Team reserves the entire class of keys that do not start with a ASCII alphabetic character in order to standardize certain parameters.

There are multiple goals for this:
* The Zenoh Team wishes to standardized some of the common use-cases it envisions for value-selectors, especially those that could be tricky to implement when considering a diverse-distributed network. Part of why standardizing them can be slow is that implementing them properly is often non-trivial, or that multiple conventions on their implementation could arise, leading to confusion. If you'd like to speed up standardization for some of the announced standardization initiatives (listed in [this folder](./)), please reach out to us.
* Some of these use-cases could benefit from `zenohd` being aware that they are being used: `_time` allows Zenoh to know you're likely addressing time-series databases, for example.
* By reserving a class of keys instead of individual ones, the Zenoh Team hopes never to break user-defined value selector keys, even when introducing new features.

Here are the currently standardized keys:
* [`_time`](./_time.md), a way to select data according to the instant it relates to.
Here are some of the initiatives we are working on:
* [`_filter`](./_filter.md), a way to select data according to predicates on its value.
* [`_project`](./_project.md), a way to only transmit part of the selected data.

### Namespace reservation
The Zenoh Team is currently considering setting up a space for developpers to share the prefix/suffix they would like to reserve as a mean of coordination for diverse networks. However, no commitment to the establishment organization space is made, and it will have to run on the honor system.
