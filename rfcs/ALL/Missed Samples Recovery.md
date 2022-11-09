# Missed Samples Recovery

## Context

The Zenoh `Subscriber` accepts a `Reliability` option. When set to `Reliable`, the Zenoh infrastructure will use a reliable Zenoh transport on each "Hop" of the routes from the several matching `Publishers` to the `Subscriber`. If the matching `Publishers` congestion control option is set to `Block`. No samples are ever dropped along those routes and back pressure is eventually applyed back to the `Publishers` in case of congestion. 

This offers reliable communications from `Publishers` to matching `Subscribers` in a stable Zenoh infrastructure. But, in case of failures (Zenoh router crash) or topology changes (redeployment of routers, etc...) some samples may be lost in the process. 

Zenoh has been designed to be highly scalable and to offer many-to-many pub/sub communications in wide systems. That's one fo the reasons why a Zenoh `Publisher` rarely knows the complete list of matching `Subscribers` in the system nor a Zenoh `Subscriber` knows it's matching `Publishers`. Setting up reliability channels for all matching `Publisher`/`Subscriber` pairs in the system would break this scalability and apply a lot of pressure on both `Publishers` and `Subscribers`.

The **Missed Samples Recovery** feature provides a way to achieve reliable communication from `Publishers` to matching `Subscribers` even in case of failures or topology changes while remaining as much scalable as possible. In particular, while the state to be maintained in `Subscribers` is still proportional to the number of matching `Publishers`, the state to be maintained in `Publishers` is independent from the number of `Subscribers`. The reliability state (cache) can even be completely removed from constrained `Publishers` and deported to key places in the system. The feature does not require the transmission of any extra control messages (AckNacks, heartBeats) in stable situation (when there is no loss). Extra messages are only required on missed samples detection.

## Basics

The Missed Samples Recovery functionnality allows to: 

* Attach `SourceInfo` metadata to published samples.
* Store published samples in a `PublicationCache` on the publisher side and/or anywhere in the network.
* Detect and query missed samples on the subscriber side.

A prototype of the functionnality is available in the `zenoh-ext` crate. It is compatible with classic Zenoh pub/sub. A `ReliableSubscriber` can receive data from a classic Zenoh `Publisher` and a classic Zenoh `Subscriber` can receive data from a `ReliablePublisher`.

## The `SourceInfo` metadata

The `SourceInfo` metadata contains: 
* a `source_id` field that uniquely identifies the source of the sample.
* a `source_sn` field that is incremented for each sample from the source.

```rust
pub struct SourceInfo {
    pub source_id: Option<ZenohId>,
    pub source_sn: Option<ZInt>,
}
```

## The `ReliablePublisher`

The `ReliablePublisher` can be built from a Zenoh `Session` by importing the `SessionExt` trait from the `zenoh-ext` crate and using it's `declare_reliable_publisher` funtion.

It embeds a classic Zenoh `Publisher` and attaches a `SourceInfo` to each published sample with:
* the `ZenohId` of the local Zenoh `Session` in the `source_id` field.
* a sequence number in the `source_sn` field that is incremented for each sample.

The `ReliablePublisher` by default creates a `ReliabilityCache` with the same key expression. But this `ReliabilityCache` can be disabled passing `false` to the `with_cache` function of the `ReliablePublisherBuilder`.
The history depthof the `ReliabilityCache` can be configured with the `history`function of the `ReliablePublisherBuilder`.

## The `ReliabilityCache`

The `ReliabilityCache` can be built from a Zenoh `Session` by importing the `SessionExt` trait from the `zenoh-ext` crate and using it's `declare_reliability_cache` funtion.

It acts similarily to a Zenoh storage. It declares a `Subscriber` for a given key expression and stores received samples. *[For now the only available implementation stores samples in memory].*

The allowed origin location of the `Subscriber` is configurable. The `ReliabilityCache` of a `ReliablePublisher`  only allows origin `Locality::SessionLocal` to only store samples from the local `ReliablePublisher` while a `ReliabilityCache` deployed somewhere alse in the system typically allows origin `Locality::Any` to store samples from all matching `ReliablePublishers`.

The `ReliabilityCache` ignores any received sample with no `SourceInfo` metadata attached to it. On reception of a sample with `SourceInfo` metadata, it creates a queue for the key expression/source_id pair with a configurable length if not already existing and pushes received sample to it. When the queue is full, oldest samples are removed from the queue on reception of new samples. The length of the queue is configurable through the `history` function of the `ReliabilityCacheBuilder`. 

The `ReliabilityCache` also creates a `Queryable` on the given key expression with a configurable prefix. This prefix should either be the id of a `Publisher` if the cache only stores samples from a given publisher or `*` if the cache stores samples from all `Publishers` (examples: `"<source_id>/key/expression"`, `"*/key/expression"`). When receiving a query, the `ReliabilityCache` extracts the prefix from the query key expression, interprets it as a `source_id` (or `*`) and replies with matching samples from the given source. If present, the `ReliabilityCache` also interprets the `_sn` query parameter as a sequence number range (in the form `"\<start\>..\<end\>"` so for examples: `"_sn=12..24"`, `"_sn=12.."`, `"_sn=..24"`) and only replies with samples which `source_sn` is in the given range.

## The `ReliableSubscriber`

The `ReliableSubscriber` can be built from a Zenoh `Arc<Session>` by importing the `ArcSessionExt` trait from the `zenoh-ext` crate and using it's `declare_reliable_subscriber` funtion.

The `ReliableSubscriber` creates a classic `Subscriber`. Samples received with no attached `SourceInfo` metadata are passed to the user. The first sample received with a given `source_id` is also passed to the user. If a sample with an already known `source_id` is received, if it's `source_sn` is the expected one (last received `source_sn` + 1) it is passed to the user, if not, the sample is stored in a pending queue and a query is issued with the `ReliableSubscriber` key expr, the `source_id` as prefix and the range of missing samples as parameter (for example: `"<source_id>/key/expression?_sn=<last_sn>..<new_sn>"`). The replies of the query are sorted and passed to the user in `source_sn` order. Then pending samples (for the given source) are passed to the user. If any samples are still missing, an error message is logged.

## Periodic queries

The behavior described in the previous section implies that a new sample has to be received to detect that some older samples have been missed. This is not necessarily a problem for periodic publications but might be for sporadic ones. To overcome this problem, the `ReliableSubscriber` can be configured to periodically send queries for eventual missed samples. The period of those queries is also configurable. 

Again, a delibereate choice has been made here to put the burden of querying for eventual missed samples on subscriber side.

## Historical data

The `ReliableSubscriber` can be configured to issue a query (`"*/key/expression"`) at it's creation to retrieve historical samples. The replies of the query are sorted and passed to the user in `source_sn` order prior to "live" samples.

## Deployment considerations

The deployment of `ReliabilityCaches` "outside" the `ReliablePublishers` can allow to desactivate the cache on the `ReliablePublisher` side and decrease the ressource pressure on them. Deploying `ReliabilityCaches` nearer to the `ReliableSubscribers` also allows to serve the `ReliableSubscribers` queries for missing samples more efficiently and more quickly.

On the other hand, it is necessary that at least one `ReliabilityCache` receives the data from any `ReliablePublisher` so that the sample is stored and eventually replied to queries. One way to ensure that is to keep the `ReliablePublishers` colocated `ReliabilityCache` enabled.

At this time, the `ReliableSubscribers` queries for missing samples are routed to the nearest `ReliabilityCache` even if this cache didn't receive some samples and have less samples than other `ReliabilityCaches` in the system. The only way to overcome this is to **only** deploy the `ReliabilityCaches` colocated to the `ReliablePublishers`.

So the deployment of `ReliabilityCaches` is a trade of between efficiency and reliability. It also depends on which failures the deployer wants to be resilient upon. If failures close to the `ReliableSubscriber` side are the ones to overcome, deploying `ReliabilityCaches` nearer to the `ReliableSubscriber` in the "stable" part of the system may be an option.

## Future improvements

### `Put` replies

There is a plan to add in Zenoh the possibility for `Subscribers` to send replies to received samples and acknowledge them. This would allow `ReliablePublishers` to know how many `ReliabilityCaches` received and stored their publication and eventually make sure that the desired number of `ReliabilityCaches` stored their publication by resending samples with the same sequence number.

### Implementation of `ReliabilityCache` by Zenoh storages

The implmentation of the `ReliabilityCache` behavior in Zenoh storages plugin would allow the `ReliabilityCache` to:
* benefit from the multiple storages back-ends (InfluxDb)
* benefit from the storages plugin alignment algorithm
