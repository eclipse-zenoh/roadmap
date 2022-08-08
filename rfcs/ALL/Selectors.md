# Selectors
## Preface
With [Key Expressions](./Key%20Expressions.md), Zenoh provides a way to act on a lot of data through a single operation.

The original goal of selectors was to allow selecting data not just through their keys, but also through their value (when the data is stored in a self-describing manner, or when the queryable is able to interpret the data).

However, its main use currently is to pass arguments to the queryables when performing requests to them.

## Syntax
The selector syntax is derived from the URL syntax, so let's start with an example:
```
https://some.hostname.com/a/path/to/something?arg1=val1&arg2=value%202
^                       ^ ^                 ^ ^                      ^
|Host: irrelevant in NDN|/|--Path, KE-like--|?|---query parameters---|
```
