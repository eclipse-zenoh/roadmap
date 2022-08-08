# Reliability in Zenoh
Zenoh supports reliability by providing hop-to-hop reliability in the general case. However, as a pub-sub network, it must contend with some challenges that typical one-to-one systems don't have to deal with.

Let's discuss one such issue as an introduction.

## How slow readers could destroy a network
It is typical to let subscribers decide what level of reliability they expect. In one-to-one channels, this is used to set up the assertion that no data loss happens, with congestion only affecting the two peers.

However, in a pub-sub network, unless some very ressource intensive measures are taken to ensure that each subscriber is distinguished from the others; an extremely slow subscriber could saturate sending-buffers all the way back to the publisher, applying backpressure onto it and reducing its publication rate. And like this, an abnormally slow subscriber could end up affecting this publisher's frequency for the whole network.

## Splitting reliability
To address the issue of slow readers hogging ressources all over the network, Zenoh takes the subscriber's demand for reliability as a request rather than an order.

Publishers publish their data with an associated Congestion Control strategy, which can be either to Block or to Drop when the network becomes too congested for data to be inserted in a buffer anymore.

If the Drop strategy has been selected, and that push comes to shove, the subscriber's expectation that a message is never dropped becomes false, as under extreme pressure, some messages may be dropped if the publisher marked them as droppable.

To avoid accidental network deadlocks, the default for publishers is to use the Drop Congestion Control strategy. However, using Block may be reasonnable if you're certain that your infrastructure will never be fully saturated.
