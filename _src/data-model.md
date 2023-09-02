# Serde data model

The Serde data model is the API by which data structures and data formats
interact. You can think of it as Serde's type system.

In code, the serialization half of the Serde data model is defined by the
[`Serializer`] trait and the deserialization half is defined by the
[`Deserializer`] trait. There is a mapping for every Rust basic data structure
into one of 29 possible types. Each method of the `Serializer` trait corresponds
to one of the types of the Serde data model.

When serializing a data structure into some format, the [`Serialize`]
implementation for the data structure is responsible for mapping that data
structure into the Serde data model by invoking exactly one of the `Serializer`
methods. Additionally, a `Serializer` implementation for the target data format is
responsible for mapping the Serde data model into its
representation.

When deserializing a data structure from a data format, the [`Deserialize`]
implementation for the data format is responsible for two things. The first is providing an implementation of [`Visitor`] to map provided portions of the input into the Serde data model per the various trait functions. The second is providing an implementation of [`Deserialize`] that invokes the appropriate method on Visitor against the appropriate portion of the input. 
Additionally, a `Deserializer` implementation for the target data structure itself
is responsible for constructing a [`Visitor`] over the input and building the data structure by passing it to functions as necessary for constructing the subcomponents of the data structure 
from the input (or remaining input). Eventually
these functions will invoke methods on the provided [`Visitor`] to consume the
portion of the input they will be provided.

[`Serializer`]: https://docs.rs/serde/1/serde/trait.Serializer.html
[`Deserializer`]: https://docs.rs/serde/1/serde/trait.Deserializer.html
[`Serialize`]: https://docs.rs/serde/1/serde/trait.Serialize.html
[`Deserialize`]: https://docs.rs/serde/1/serde/trait.Deserialize.html
[`Visitor`]: https://docs.rs/serde/1/serde/de/trait.Visitor.html

## Types

The Serde data model is a simplified form of Rust's type system. It consists of
the following 29 types:

- **14 primitive types**
  - bool
  - i8, i16, i32, i64, i128
  - u8, u16, u32, u64, u128
  - f32, f64
  - char
- **string**
  - UTF-8 bytes with a length and no null terminator. May contain 0-bytes.
  - When serializing, all strings are handled equally. When deserializing, there
    are three flavors of strings: transient, owned, and borrowed. This
    distinction is explained in [Understanding deserializer lifetimes] and is a
    key way that Serde enabled efficient zero-copy deserialization.
- **byte array** - [u8]
  - Similar to strings, during deserialization byte arrays can be transient,
    owned, or borrowed.
- **option**
  - Either none or some value.
- **unit**
  - The type of `()` in Rust. It represents an anonymous value containing no
    data.
- **unit_struct**
  - For example `struct Unit` or `PhantomData<T>`. It represents a named value
    containing no data.
- **unit_variant**
  - For example the `E::A` and `E::B` in `enum E { A, B }`.
- **newtype_struct**
  - For example `struct Millimeters(u8)`.
- **newtype_variant**
  - For example the `E::N` in `enum E { N(u8) }`.
- **seq**
  - A variably sized heterogeneous sequence of values, for example `Vec<T>` or
    `HashSet<T>`. When serializing, the length may or may not be known before
    iterating through all the data. When deserializing, the length is determined
    by looking at the serialized data. Note that a homogeneous Rust collection
    like `vec![Value::Bool(true), Value::Char('c')]` may serialize as a
    heterogeneous Serde seq, in this case containing a Serde bool followed by a
    Serde char.
- **tuple**
  - A statically sized heterogeneous sequence of values for which the length
    will be known at deserialization time without looking at the serialized
    data, for example `(u8,)` or `(String, u64, Vec<T>)` or `[u64; 10]`.
- **tuple_struct**
  - A named tuple, for example `struct Rgb(u8, u8, u8)`.
- **tuple_variant**
  - For example the `E::T` in `enum E { T(u8, u8) }`.
- **map**
  - A variably sized heterogeneous key-value pairing, for example `BTreeMap<K,
    V>`. When serializing, the length may or may not be known before iterating
    through all the entries. When deserializing, the length is determined by
    looking at the serialized data.
- **struct**
  - A statically sized heterogeneous key-value pairing in which the keys are
    compile-time constant strings and will be known at deserialization time
    without looking at the serialized data, for example `struct S { r: u8, g:
    u8, b: u8 }`.
- **struct_variant**
  - For example the `E::S` in `enum E { S { r: u8, g: u8, b: u8 } }`.

[Understanding deserializer lifetimes]: lifetimes.md

## Mapping into the data model

In the case of most Rust types, their mapping into the Serde data model is
straightforward. For example the Rust `bool` type corresponds to Serde's bool
type. The Rust tuple struct `Rgb(u8, u8, u8)` corresponds to Serde's tuple
struct type.

But there is no fundamental reason that these mappings need to be
straightforward. The [`Serialize`] and [`Deserialize`] traits can perform *any*
mapping between Rust type and Serde data model that is appropriate for the use
case.

As an example, consider Rust's [`std::ffi::OsString`] type. This type represents
a platform-native string. On Unix systems they are arbitrary non-zero bytes and
on Windows systems they are arbitrary non-zero 16-bit values. It may seem
natural to map `OsString` into the Serde data model as one of the following
types:

- As a Serde **string**. Unfortunately serialization would be brittle because an
  `OsString` is not guaranteed to be representable in UTF-8 and deserialization
  would be brittle because Serde strings are allowed to contain 0-bytes.
- As a Serde **byte array**. This fixes both problems with using string, but now
  if we serialize an `OsString` on Unix and deserialize it on Windows we end up
  with [the wrong string].

Instead the `Serialize` and `Deserialize` impls for `OsString` map into the
Serde data model by treating `OsString` as a Serde **enum**. Effectively it acts
as though `OsString` were defined as the following type, even though this does
not match its definition on any individual platform.

```rust
# #![allow(dead_code)]
#
enum OsString {
    Unix(Vec<u8>),
    Windows(Vec<u16>),
    // and other platforms
}
#
# fn main() {}
```

The flexibility around mapping into the Serde data model is profound and
powerful. When implementing `Serialize` and `Deserialize`, be aware of the
broader context of your type that may make the most instinctive mapping not the
best choice.

[`std::ffi::OsString`]: https://doc.rust-lang.org/std/ffi/struct.OsString.html
[the wrong string]: https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/
