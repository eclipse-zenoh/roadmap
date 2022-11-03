# Consolidation
## Preface
Zenoh being distributed by nature, any query (performed through the `get` operation) is susceptible to receive multiple responses sharing the same key. However, queriers are often only interested in the latest value for each key in the response, and are only interested in receiving it once.

We call the process of ignoring duplicate and/or obsolete values for a same key *consolidation*. To avoid each querier re-implementing it, as well as to provide efficiencies that couldn't be applied by a querier otherwise, Zenoh provides it through *consolidation modes*.

## Consolidation Modes
### None
Consolidation can be fully disabled, letting every queryable's every reply reach the querier.

### Latest
This is the strongest consolidation: received samples are held back until every queryable has replied, keeping only the sample with the highest timestamp for each key.

Note that since samples are held back until every queryable has answered, this introduces latency; and stores the samples in memory in the meantime, which may require large amounts of RAM depending on circumstances.

### Monotonic
This is a compromise between the *None* and *Latest* modes: when a sample is received, its timestamp is checked against the one that may have been previously stored for that sample's key:
- if no such timestamp exists, the received one is stored and the sample is immediately forwarded.
- if the stored timestamp is lower than the new sample's one, the stored timestamp is replaced and the sample is immediately forwarded.
- if the stored timestamp is higher or equal, the sample is dropped.

This provides low latency, while ensuring that any received sample is the latest observed *yet*. Since only key-timestamp pairs are stored for consolidation, this is less memory-intensive than the *Latest* mode. Note however that this doesn't guarantee that your querier only obtain a single response per key.

Note that depending on the receive order, the number of received samples for the same key in the same system may vary.

## Consolidation Locations
Currently, consolidation only happens on the querier device. However, the Zenoh Team plans to allow Zenoh routers to perform part of the consolidation effort, notably as a means to save bandwidth by avoiding the forwarding of values that aren't actually of interest to the querier.

The means by which the user may specify *where* consolidation happens are still up for debate within the team.

## Automatic consolidation
The user may also *not* select any consolidation mode, which some languages may expose as a fake `Auto` mode, while others may simply expose the mode as an optional value and consider the absence of a value the selection of the automatic mode.

Automatic consolidation mode will resolve to the *Latest* mode, unless the selector for the query contains a [`_time`](../Selector/_time.md) argument, which hints at a time-series query which will use the *None* mode instead.

The automatically selected modes are liable to change on updates of the client library, but should only do so to provide more garantees.

By default, queries will use automatic consolidation mode selection.
