# The `_time` argument and Zenoh Time DSL
The `_time` argument is used to select only samples that are timestamped within a range passed as its value.

This is notably useful when your infrastructure includes time-series databases, such as [InfluxDB](https://github.com/eclipse-zenoh/zenoh-backend-influxdb), as part of its storages.

## Syntax

The structural representation of the Zenoh Time DSL, which may adopt one of two syntax:
- the "range" syntax: `<ldel: '[' | ']'><start: TimeExpr?>..<end: TimeExpr?><rdel: '[' | ']'>`
- the "duration" syntax: `<ldel: '[' | ']'><start: TimeExpr>;<duration: Duration><rdel: '[' | ']'>`, which is
  equivalent to `<ldel><start>..<start+duration><rdel>`

Durations follow the `<duration: float><unit: "u", "ms", "s", "m", "h", "d", "w">` syntax.

Where `TimeExpr` itself may adopt one of two syntaxes:
- the "instant" syntax, which must be a UTC [RFC3339](https://datatracker.ietf.org/doc/html/rfc3339) formatted timestamp.
- the "offset" syntax, which is written `now(<sign: '-'?><offset: Duration?>)`, and allows to specify a target instant as
  an offset applied to an [instant of evaluation](#instant-of-evaluation).

In range syntax, omiting `<start>` and/or `<end>` implies that the range is unbounded in that direction. Keep in mind that some queryables might peek into the future if unbounded (imagine a weather recording and predicting queryable).

Exclusive bounds are represented by their respective delimiters pointing towards the exterior.
Interior bounds are represented by the opposite.

The comparison step for instants is the nanosecond, which makes exclusive and inclusive bounds extremely close.
The `[<start>..<end>[` pattern may however be useful to guarantee that a same timestamp never appears twice when
iteratively getting values for `[t0..t1[`, `[t1..t2[`, `[t2..t3[`...

## Instant of evaluation
The offset syntax is somewhat ambiguous as to when `now` is.

The current consensus is that `now` is resolved on queryable-side, as late as possible (in the case of InfluxDB, the `now` is even transmitted to the DB for evaluation by InfluxDB). This is for two reasons:
* `now`'s principal use-case is to address data that's as recent as possible, while keeping the guarantee not to peer into the future.
* The alternative is resolving `now` on querier side, which can be achieved by other means. Rust code for example can construct a `TimeRange<TimeExpr>`, resolve it into a `TimeRange<SystemTime>`, which will convert the `now` expressions into instant syntax once re-serialized via the `std::fmt::Display` trait.
