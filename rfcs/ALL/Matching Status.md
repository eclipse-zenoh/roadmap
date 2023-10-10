# Publisher Matching Status

> 🔬 **Unstable**<br/>
> The API defined in this document is **unstable**. It may change it in a future release.

## Context

It is sometimes useful to know if there exist Subscribers matching a given key expression or not. For example it avoids computing data before publishing it when this computation is costly.

[Liveliness](Liveliness) is a way to achieve this. But it's more costly than strictly necessary and implies tokens declarations on the subscriber side.

One of the roles of a Zenoh Publisher is to perform writer side filtering. To perform such writer side filtering the Zenoh local infrastructure (library) has to know if there exist matching Subscribers or not. So it would make sense to give user access to this information from a Publisher.

## The MatchingStatus struct

The `MatchingStatus` struct represents the matching status of a `Publisher` at a given point in time: it indicates if there exist Subscribers matching the Publisher’s key expression or not.

It provides a single function that returns `true` if there exist Subscribers matching the Publisher’s key expression and `false` otherwise.

```rust
pub fn is_matching(&self) -> bool
```

Note 1: Using an opaque struct with a function rather than a `bool` allows potential backward compatible evolutions in the future.

Note 2: If we already want to prepare for the eventual situation in which Zenoh couldn't know if there exist matching Subscribers or not, we could rename the function:

```rust
pub fn maybe_matching(&self) -> bool
```

Or provide both: 
```rust
pub fn is_matching(&self) -> ZResult<bool>
pub fn maybe_matching(&self) -> bool
```

## Accessing the MatchingStatus

The `Publisher` provides the following function: 
```rust
pub fn matching_status(&self) -> impl Resolve<ZResult<MatchingStatus>>
```

**Example:**
```rust
use zenoh::prelude::r#async::*;

let session = zenoh::open(config::peer()).res().await.unwrap().into_arc();
let publisher = session.declare_publisher("key/expression").res().await.unwrap();
let is_matching: bool = publisher.matching_status().res().await.unwrap().is_matching();
```

## Listening to MatchingStatus changes

A `MatchingListener` may be created from a `Publisher` that will send notifications when the MatchingStatus of a the publisher changes. The `MatchingListener` is similar to a `Subscriber` and allows to receive `MatchingStatus` through callbacks or streams.

The `Publisher` provides the following function: 
```rust
pub fn matching_listener(&self) -> MatchingListenerBuilder<'_, DefaultHandler>
```

**Example:**
```rust
use zenoh::prelude::r#async::*;

let session = zenoh::open(config::peer()).res().await.unwrap();
let publisher = session.declare_publisher("key/expression").res().await.unwrap();
let matching_listener = publisher.matching_listener().res().await.unwrap();
while let Ok(matching_status) = matching_listener.recv_async().await {
    if matching_status.is_matching() {
        println!("Publisher has matching subscribers.");
    } else {
        println!("Publisher has NO MORE matching subscribers.");
    }
}
```

## Alternatives

Some proposed this alternative naming:
```rust
impl Publisher {
    pub fn matching_subscribers(&self) -> impl Resolve<ZResult<MatchingSubscribers>>
    pub fn matching_subscribers_listener(&self) -> impl  Resolve<ZResult<MatchingSubscribersListener>>
}
```

```rust
impl MatchingSubscribers {
    pub fn exist(&self) -> bool
}
```
or:
```rust
impl MatchingSubscribers{
    pub fn exist(&self) -> ZResult<bool>
    pub fn may_exist(&self) -> bool
}
```
