# Query Payload

The ability to send arbitrarily large data with a query has emerged from multiple use cases. To fullfill this need, Zenoh provides a way to optionally attach a `Value` (Encoding and Payload) to sent queries.

## The wire protocol

The `Query` message has a flag `B` that indicates the presence or absence of a `QueryBody`.
```
 7 6 5 4 3 2 1 0
+-+-+-+-+-+-+-+-+
|K|B|T|  QUERY  |
+-+-+-+---------+
~    KeyExpr     ~ if K==1 then key_expr has suffix
+---------------+
~selector_params~
+---------------+
~      qid      ~
+---------------+
~     target    ~ if T==1
+---------------+
~ consolidation ~
+---------------+
~   QueryBody   ~ if B==1
+---------------+
```

The `QueryBody` contains a `DataInfo` and a `Payload`: 
```
 7 6 5 4 3 2 1 0
+---------------+
~    DataInfo   ~
+---------------+
~    Payload    ~
+---------------+
```

The `DataInfo` optionally contains more fields than the needed `Encoding` in this context. This allows the API to be enriched with no protocol change in the future.

## The API

### Rust

The `GetBuilder` has 2 functions that allow to attach a `Value` to the query: 

```rust
pub fn with_value<IntoValue>(mut self, value: IntoValue) -> Self
where IntoValue: Into<Value>;
```

The `Query` struct (received by `Queryables`) has a function to access the optionally attached `Value`:

```rust
pub fn value(&self) -> Option<&Value>;
```
