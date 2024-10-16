# Zenoh serialization format

Zenoh is an untyped protocol which manipulates raw bytes. It's possible to use Zenoh with any kind of serialization format, like JSON, Protobuf, Cap'n Proto, etc.

For convenience, Zenoh also provides a serialization implementation as an extension. The serialization format is primarily intended to be simple, in order to be easily supported in all Zenoh bindings, with a secondary goal of enabling efficient implementations for most platforms.

However, it should be reminded that Zenoh serialization format is only one format among others, and that Zenoh will work the same independently of the serialization format chosen. Other formats may provide better performance, smaller serialized size, or other advantages, so the decision about format should be done consciously.

Zenoh serialization format is not self-describing, it requires to know the schema used to encode data in order to decode them. The format is specified in the following sections of the document; examples are written with the following scheme:
```
<data with rust notation> --> <hex representation of serialized data>
```

### Numbers

Numbers are serialized with their fixed-size little endian representation; signed integers use two's complement, and floating-point numbers use IEEE 754.

Boolean is a special case of 8bit unsigned integer, with value restricted to either 0 or 1.

```
7u8    --> 0x07
-1i8   --> 0xff
0i32   --> 0x00000000
42i32  --> 0x0000002a
1.5f32 --> 0x3fc00000
true   --> 0x01
```

### Sequences

Sequences, i.e. variable-size repetitions of items with the same serialization schema, are serialized by first writing the sequence length using LEB128 encoding, then concatenating the serialization of each of its items.

Because some languages supported in Zenoh bindings don't provide a fixed-size array type, e.g. Python, languages providing it like Rust should implement serialization of fixed-size array as a variable-length sequence. It allows deserializing a `[f64; 3]` from Rust as a `list[float]` in Python. Map should be serialized as a sequence of 2-tuples.

```
[1u8, 2, 3] --> 0x03010203
```

### Strings

Strings are a particular kind of sequence, composed of UTF-8 bytes, which doesn't include a NULL terminator. They are thus serialized by first writing the string length using LEB128 encoding, then the UTF-8 bytes of the string.
 
Bindings for which the language provides a "string" type should implement its serialization as a UTF-8 byte sequence.

```
"Hello!" --> 0x0648656c6c6f21
```

### Tuples

Tuples are serialized as the concatenation of the serializations of each of its items.

```
(42u8, 0.5f32)               --> 0x2a3f000000
((42u8, 0.5f32), false)      --> 0x2a3f00000000
[(0, "hello"), (1, "world")] --> 0x02000568656c6c6f0105776f726c64
```

