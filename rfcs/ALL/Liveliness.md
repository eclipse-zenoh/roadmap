# Liveliness

> 🔬 **Unstable**<br/>
> The API, resource keys and value formats defined in this document are **unstable**. They may change it in a future release.

## Context

The ability to check the liveliness of a Zenoh entity (application, publisher, subscriber, etc...) and the ability to be notified of liveliness changes of Zenoh entities (apparition/disappearance of an entity) is a functionnality that is needed in some use cases.

The ability to monitor the liveliness of any entity in the system at any time puts a lot of pressure on the infrastructure. Zenoh needs to anticipate the possibility that any application could monitor the liveliness of any other entity at any time even if this never happens. This will cause huge scalability issues in a large system with a lot of entities.

To offer such functionality without impacting scalability, users should be able to explicitely declare which entities may have their liveliness monitored by other applications or not.

Such a functionality is already offered by the group management feature in `zenoh-ext`, but the group management feature is quite resource consuming and may also suffer scalability issues. In particular each group member needs to periodically assert it's liveliness by broadcasting a HeartBeat message to all other members of the group.

It should be possible to build a far less resource consuming generic liveliness mechanism in the routing layer that would rely on the existing *Transport Session* liveliness.

## The basics

Any Zenoh application can declare one or multiple liveliness tokens. Each liveliness token is declared on a key expression. A declared liveliness token will be seen as *alive* by any other Zenoh application in the system that monitors it while the liveliness token is not undeclared or dropped, while the Zenoh application that declared it is *alive* (didn't stop or crashed) and while the Zenoh application that declared the token has Zenoh connectivity with the Zenoh application that monitors it. 

In the case of a network partition, the Zenoh applications that still have connectivity with the Zenoh application that declared the token still see it as *alive* while the Zenoh applications that loosed connectivity with the Zenoh application that declared the token see it as *dropped*.

## Multiple liveliness tokens on the same key

If one or multiple Zenoh applications declare multiple liveliness tokens on the same key, the token will be seen as *alive* as soon as the first declaration occurs and will be seen as *dropped* when the last token is dropped. 

## The Liveliness struct

All liveliness related functions are gathered in a `Liveliness` struct that can be accessed through a `liveliness`funtion on the Zenoh `Session.

**Example:**
```rust
let session = zenoh::open(Config::default()).res().unwrap();
let liveliness = session.liveliness();
```

## Declaring liveliness tokens

A liveliness token can be declared on any key expression with the help of the `declare_liveliness` function of the `Liveliness` struct:

```rust
pub fn declare_liveliness<'a, 'b, TryIntoKeyExpr>(
    &'a self,
    key_expr: TryIntoKeyExpr,
) -> LivelinessTokenBuilder<'a, 'b>;
```

**Example:**
```rust
let session = zenoh::open(Config::default()).res().unwrap();
let token = session
    .liveliness()
    .declare_liveliness("group1/member1")
    .res()
    .unwrap();
```

A liveliness token can be explicitely undeclared or dropped.

**Example:**
```rust
let session = zenoh::open(Config::default()).res().unwrap();
let token = session
    .liveliness()
    .declare_liveliness("group1/member1")
    .res()
    .unwrap();
token.undeclare();
```

## Querying liveliness tokens

Each *alive* token can be accessed through queries at any pont in the system with the help of the `get` funtion of the `Liveliness`struct:

**Example:**
```rust
let session = zenoh::open(Config::default()).res().unwrap();
let tokens = session
    .liveliness()
    .get("group1/*")
    .res()
    .unwrap();
while let Ok(token) = tokens.recv() {
    match token.sample {
        Ok(sample) => println!("Alive token ('{}')", sample.key_expr.as_str(),),
        Err(err) => println!("Received (ERROR: '{}')", String::try_from(&err).unwrap()),
    }
}
```

## Notifications of liveliness changes

The liveliness changes can be monitored with the help of the `declare_subscriber` function of the `Liveliness` struct.

Each time a new liveliness token is declared a matching liveliness subscriber will receive a `Sample` with kind `Put`.

Each time a liveliness token is *dropped* (because the application that declared it stopped, crashed or loosed connectivity), a matching liveliness subscriber will receive a `Sample` with kind `Delete`.

**Example:**
```rust
let session = zenoh::open(Config::default()).res().unwrap();
let subscriber = session
    .liveliness()
    .declare_subscriber("group1/*")
    .res()
    .unwrap();
while let Ok(change) = subscriber.recv() {
    match change.kind {
        SampleKind::Put => println!(
            "Alive token ('{}')", change.key_expr.as_str()),
        SampleKind::Delete => println!(
            "Dropped token ('{}')", change.key_expr.as_str()),
    }
}
```

## Using the `QueryingSubscriber`

When monitoring liveliness changes by declaring a `Subscriber`, the `Subscriber` only receives changes that occured after it's declaration. In particular, the `Subscriber` will not see liveliness tokens declared before the `Subscriber` declaration. In order to access liveliness tokens declared before the `Subscriber` declaration as well as future connectivity changes, a `zenoh-ext` `QueryingSubscriber` can be used.

**Example:**
```rust
let session = zenoh::open(Config::default()).res().unwrap();
let subscriber = session
    .liveliness()
    .declare_subscriber("group1/*")
    .querying()
    .res()
    .unwrap();
while let Ok(change) = subscriber.recv() {
    match change.kind {
        SampleKind::Put => println!(
            "Alive token ('{}')", change.key_expr.as_str()),
        SampleKind::Delete => println!(
            "Dropped token ('{}')", change.key_expr.as_str()),
    }
}
```

## Client liveliness Subscribers loosing connectivity

When a client Zenoh application that subscribed to liveliness changes looses connectivity with the system (looses connectivity with it's router and fails to reesatblish connectivity), as it loosed connectivity with all remote applications a user could expect to receive a `delete` for all existing liveliness tokens. But this implies keeping a state of all tokens in the client itself which is undesirable.

So, in such situation, to indicate to the user that all tokens are *dropped*, the following `delete` is sent to matching liveliness subscribers:
- key: `**`
- kind: `Delete`
- value: empty

## Future improvements

### Liveliness tokens associated value

It may be usefull to allow Zenoh applications to attach a `value` to liveliness tokens they declare. This attached `value` would be seen by applications that query or subscribe to liveliness tokens/changes.

The `LivelinessTokenBuilder` would offer a `with_value` funtion: 

```rust
pub fn with_value<IntoValue>(mut self, value: IntoValue) -> Self
where IntoValue: Into<Value>;
```

**Example:**
```rust
let session = zenoh::open(Config::default()).res().unwrap();
let token = session
    .declare_liveliness("group1/member1")
    .with_value('value')
    .res()
    .unwrap();
```

**Multiple associated values on the same key**

This rises the question of which value is seen associated to a token when the token has been declared multiple times on the same key with different values. Two behaviors could be offered in such case: 
- **Undefined behavior:** queriers/subscribers will receive any of the associeted values.
- **Last value behavior:** queriers/subscribers receive the latest associated value. This implies that tokens declaration are timestamped and implies more traffic as each time a token is declared again with the same key it needs to be repropagated in the whole system.