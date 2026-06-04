- Feature Name: `ff_prime_field_repr_endianness`
- Start Date: 2026-06-04
- RFC PR: [zkcrypto/rfcs#4](https://github.com/zkcrypto/rfcs/pull/4)
- Tracking Issue: [zkcrypto/group#78](https://github.com/zkcrypto/group/issues/78)

# Summary
[summary]: #summary

Add an enum with variants for distinguishing little versus big endian to the `ff` crate, as well
as an associated constant value of this type to the `PrimeField` trait, in order to handle cases
where curves use a big endian rather than little endian scalar serialization.

# Motivation
[motivation]: #motivation

Almost all the elliptic curve implementations maintained by the @RustCrypto organization use the
SEC1 serialization for field elements, which is big endian.

This primarily becomes a problem when trying to combine scalars using a big endian serialization
for `PrimeField::Repr` with the `group` crate's w-NAF implementation, which assumes a little endian
ordering despite the documentation for `PrimeField::to_repr` stipulating that "The endianness of the
byte representation is implementation-specific. Generic encodings of field elements should be
treated as opaque."

Downstream consumers of `PrimeField::Repr`, including the `group` crate, need to instead handle it
in an endian-aware manner.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Adds a new `enum` to the public API of `ff` with the following shape (e.g. inspired by the
`ff_derive` crate):

```rust
#[derive(Clone, Copy, Debug, Default, Eq, PartialEq)]
pub enum ReprEndianness {
    Big,
    #[default]
    Little,
}
```

An associated constant added to `PrimeField`, with a default of `ReprEndianness::Little` in order
to ensure the addition is non-breaking, can now be used to determine how to iterate over the bits
of `PrimeField::Repr`:

```rust
pub trait PrimeField {
    type Repr: ...;
  
    const REPR_ENDIANNESS: ReprEndianness = ReprEndianness::Little;
}
```

Any downstream code consuming `PrimeField::Repr` in a non-opaque manner will need to consult this
constant when attempting to iterate over its bytes.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The w-NAF implementation in `group` can now consult this constant when e.g. constructing
`LimbBuffer` in order to determine the order in which to iterate over the bytes of the serialized
scalar.

Nothing else in the API needs to change besides this: downstream users of the w-NAF implementation
will now have it automatically work when they use it in conjunction with an elliptic curve whose
scalars use a big endian serialization, once the curve implementation sets the constant to
`ReprEndianness::Big` when appropriate.

# Drawbacks
[drawbacks]: #drawbacks

The main drawback of this proposed approach is it leaves the door open to potential misuse where
the associated `const` for `ReprEndianness` isn't checked/honored and code continues to assume that
`PrimeField::Repr` is always little endian.

It doesn't present a single API that makes it easy to iterate over the bits of a field element
irrespective of the endianness in which `Repr` is serialized, though such an API could potentially
be added in a followup. This proposal deliberately sidesteps defining such an API in order to keep
the scope and code surface as simple as possible.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Without some way for curve implementations to signal the endianness of `PrimeField::Repr` it's not
possible to use curves with a big endian serialization thereof with generic code that needs to
iterate over the bits of a field element.

The main alternative would be to define an API that allows for iterating over the bits of a
serialized field element in least-to-most significant order, such as `PrimeField::to_le_bits`,
returning a type bounded by something like `(Into)Iterator<Item = bool>`, e.g. a new public iterator
type adapted from e.g. `LimbBuffer` that can be used generically over curves, a new associated
type, or potentially using RPIT such that the type can be completely opaque.

Such an API can still be added in a followup, and defined generically by having it automatically
consult the associated `const` for the endianness.

# Prior art
[prior-art]: #prior-art

Two places we see prior art for at least the `enum` portion of this proposal:

- `ff_derive`: has an `enum ReprEndianness`, unfortunately due to the nature of proc macro crates
  it will always need to define its own `enum` separate from everything else.
- `primefield`: @RustCrypto's generic implementation of prime fields which implement the traits
  from `ff` defines an `enum ByteOrder` with `LittleEndian` and `BigEndian` variants, but can switch
  to the `enum` from `ff` when added to the public API.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The main unresolved question is how to present an API which can consult the associated `const` which
defines the `ReprEndianness` and from it provide consistent behavior to downstream users, such that
downstream code doesn't need to check it and can always e.g. iterate over the bits of a field
element in least-to-most significant order regardless of the field element serialization,.

# Future possibilities
[future-possibilities]: #future-possibilities

The main future possibility, as sketched out in the "Rationale and alternatives" section, is to
provide a higher-level API which abstracts over the endianness so downstream code doesn't need to
be aware of it at all.

This would be beneficial for preventing misuses where the endianness is incorectly assumed to always
be little endian.
