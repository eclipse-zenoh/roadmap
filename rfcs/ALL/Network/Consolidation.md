# Consolidation
## Preface
Zenoh being distributed by nature, any query (performed through the `get` operation) is susceptible to receive multiple responses sharing the same key. However, queriers are often only interested in the latest value for each key in the response, and are only interested in receiving it once.

We call the process of ignoring duplicate and/or obsolete values *consolidation*. To avoid each querier re-implementing it, as well as to provide efficiencies that couldn't be applied by a querier otherwise, Zenoh provides the means for consolidation via the *Consolidation Plan*, where a *Mode* is selected, and *Strategies* are chosen for each possible consolidation *Locations*.

## Consolidation Mode
The consolidation mode describes the goal of the consolidation: whether to eliminate only duplicates (samples that share the same key and timestamp) with *duplicate removal*, or also outdated values with *outdated removal* (which includes duplicate removal).

Zenoh also allows you to select neither mode (by selecting *automatic*), letting Zenoh analyze the query's [selector](../Selector) for hints as to which consolidation mode to use.

## Consolidation Strategies
Depending on the consolidation mode, various strategies for consolidation can be adopted at any consolidation location.

### Outdated removal strategies
Each location can apply one of 3 strategies to apply outdated removal:
- *Full* consolidation: the most memory intensive strategy, it stores one sample for each key where a response appears. If a response with a more recent sample pair appears for a stored key, it replaces the previously stored one. It is otherwise dropped. Once all queryables have sent a termination response or have timed out, the remaining stored samples are forwarded. Keep in mind that this consolidation may cause high latency as the slowest queryable will throttle the query.
- *None*: this disables consolidation on the selected node: any received sample is immediately forwarded.
- *Lazy* consolidation: a middle ground between *None* and *Full*. For each received sample:
  - if no sample with this key has been received yet, the key and timestamp are stored and the sample is immediately forwarded,
  - otherwise, if the sample has an older timestamp than the recorded one for its key, it is dropped. If its timestamp is newer, it is immediately forwarded and its timestamp replaces the previous one.

Lazy consolidation is generally a good middle ground, as it may reduce the network load, but doesn't affect latency as much as Full consolidation. 

Note that outdate removal treats only cares for equality of keys: if sample A's key expression includes sample B's, and is more recent than B, B will still be sent.

### Duplicate removal strategies
This consolidation can either be enabled or disabled at any of the consolidation locations.

If enabled, any received sample will be forwarded immediately, unless its key-timestamp pair was stored during a previous forward. It is useful when querying multiple time-series queryables that may return identical samples to save bandwidth.

## Consolidation Locations
3 locations are defined for use in consolidation plans:
- *Reception*: the querying device itself. Use this to reduce the impact of consolidation on the infrastructure, at the cost of higher cpu and memory usage on the querying device.
- *Last router*: the router closest to the querying device. Use this to offload the impact of consolidation to the edge of the infrastructure when you reception device might be too limited. 
- *First routers*: Each router closest to one of the queryables will attempt consolidation. This is useful if a router with sufficient memory acts as a hub between a multiple queryables.

## Default behaviour
Currently, *automatic* resolves to *full outdated removal* unless the selector contains a [`_time`](../Selector/_time.md) argument, which hints at a time-series query which will be consolidated with *duplicate removal* instead.
