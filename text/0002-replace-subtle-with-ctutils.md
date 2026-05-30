- Feature Name: `replace_subtle_with_ctutils`
- Start Date: 2026-04-07
- RFC PR: TBD
- Tracking Issue: TBD

# Summary
[summary]: #summary

This proposal is to replace the unmaintained `subtle` dependency which provides an assortment of
constant-time operations with the `ctutils` crate which has been developed
[under the RustCrypto organization](https://github.com/RustCrypto/utils/tree/master/ctutils).

# Motivation
[motivation]: #motivation

Though `subtle` has largely served the Rust ecosystem well, its maintenance status has become
problematic, with a [large number of open PRs](https://github.com/dalek-cryptography/subtle/pulls)
and no one to review them. This leaves `subtle` in a state where if there are problems it may be
difficult to address them, and it won't be able to take advantage of
[new Rust features](https://blog.trailofbits.com/2025/12/02/introducing-constant-time-support-for-llvm-to-protect-cryptographic-code/).

The strategy it implements to impede optimizations, an off-label usage of `ptr::read_volatile`, was
once the best option available on stable Rust. However, Rust itself has moved on, adding both stable
inline assembly which can be used to guarantee operations will not be removed by the compiler or
rewritten with branches, as well as `const fn` which enables functions that can potentially run at
compile-time, which is useful for calculating constants at compile-time. As an example use case, 
the RustCrypto `primefield` crate
[uses `const fn` to compute the constants for `ff::PrimeField` at compile time](https://github.com/RustCrypto/elliptic-curves/blob/0732c4a/primefield/src/monty.rs#L452-L481),
effectively providing similar functionality to `ff-derive` without the use of a proc macro.

`ctutils` got its start as an extraction of the RustCrypto `crypto-bigint` crate, where we were
using `subtle` but effectively built an entirely parallel set of `ConstChoice` and `ConstCtOption`
types which we used to implement the core of algorithms as `const fn`.

It's built on the RustCrypto [`cmov` crate](https://github.com/RustCrypto/utils/tree/master/cmov)
which uses `asm!` to either invoke specific predication intrinsics (CMOV family on x86, CSEL on ARM)
on supported platforms, or otherwise falls back to a bitwise masking-based implementation similar to
`subtle`, but further augmented on certain platforms with `asm!` mask generation which both avoids
compiler optimizations (some of which have previously actually inserted branches) and for improving
performance.

`ctutils` also removes a number of problematic bounds imposed by `subtle` like `Copy` and `Default`
which meant its traits couldn't be used with heap-allocated types, which we needed to implement
crates like `rsa`, `dsa`, and `srp`.

We recently fully migrated `crypto-bigint` into a state where `ctutils` natively provided the
`Choice` and `CtOption` type used throughout the library, but it still optionally impls relevant
`subtle` traits. `ctutils` itself also features `subtle` interop for enabling incremental migrations
which allows the `Choice` and `CtOption` types to be bidirectionally converted between the two
libraries. Our `elliptic-curve` crate has leveraged this to use `crypto-bigint` internally while
still using `subtle` in its public API. However, that still leaves us in a state where we have two
constant-time libraries, and `subtle`-based APIs can't be used in `const fn`.

The current plan is to [migrate all of the RustCrypto crates to `ctutils`](https://github.com/RustCrypto/traits/issues/2275)
so it will likely already become a dependency to other projects as some point. We have PRs open to
give a similar treatment to all of our elliptic curve crates that we did to `crypto-bigint`:
natively migrating to `ctutils` but continuing to support `subtle` trait impls as an optional
dependency. The one place where we still continue to have `subtle` in our public API though is
through the `ff` and `group` traits, particularly ones which return `subtle::Choice` or
`subtle::CtOption`, e.g. `Field::is_zero`, `Field::invert`.

And so, this proposal is to migrate `ff` and `group` to use `ctutils` instead of `subtle`. Like in
the RustCrypto crates, `subtle` support could be optionally retained to some degree, but with
the aforementioned APIs migrated to use `ctutils::Choice` and `ctutils::CtOption`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

`ctutils` as a constant-time library is largely `subtle`-shaped and should be fairly intuitive to
anyone who has previously used `subtle`. Its API primarily consists of:

- `Choice`: substitute for `bool` which internally wraps a `u8`. Accessing its inner value uses
  `hint::black_box` which (despite some scary language in its docs) should be at least as good or
  better than the `ptr::read_volatile` approach used by `subtle` (though `subtle` offers optional
  `black_box` support). Note that `ctutils` uses `black_box` in a belt-and-suspenders manner, with
  the real security provided by `cmov`. `ctutils::Choice` also offers a number of `const fn`
  constructors which can perform various constant-time operations using bitwise masking.
- `CtOption`: substitute for `Option` which evaluates its combinators eagerly and unconditionally
  rather than lazily. Unlike `subtle`, `ctutils::CtOption` can be used with non-`Copy` types.
- Traits for constant-time operations:
  - `CtAssign`: conditional constant-time assignment, equivalent to
    `subtle::ConditionallySelectable::conditional_assign`, but split out to permit impls on
    DSTs that can't otherwise be returned from a select operation.
  - `CtEq`: constant-time equality testing, equivalent to `subtle::ConstantTimeEq`
  - `CtGt`/`CtLt`: constant-time greater/less, equivalent to `subtle::ConstantTimeGreater` and
    `subtle::ConstantTimeLess`
  - `CtNeg`: constant-time conditional negation, equivalent to `subtle::ConditionallyNegatable`
  - `CtSelect`: constant-time selection, equivalent to `subtle::ConditionallySelectable` sans the
    `conditional_assign` method.

## Migration

Migrating from `subtle` to `ctutils` is a largely mechanical operation that can be done through a
series of find-and-replace operations, as documented in the
[`subtle` migration guide for `ctutils`](https://docs.rs/ctutils/latest/ctutils/#subtle-migration-guide).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO

# Drawbacks
[drawbacks]: #drawbacks

The Rust cryptography ecosystem has largely aligned on `subtle`. Switching to `ctutils` has the
unfortunate effect of splitting the ecosystem. While the interop support provides a workaround to
bridge the two ecosystems, it's certainly not as nice as having a single crate everyone uses.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The main alternative is remaining on `subtle`, either in its current 2.x form, or trying to bring
the features and functionality of `ctutils` to `subtle` in the form of non-breaking or breaking
changes.

The number of breaking changes to `subtle` that would be required to have equivalent functionality
to `ctutils` is incredibly small and likely would have little to no impact. These have been proposed
in a number of issues and pull requests, which I'll deliberately list in reverse order here as
the most recent are probably the most relevant:

- [#148: Proposed breaking changes for a subtle v3.0.0](https://github.com/dalek-cryptography/subtle/issues/148)
- [#137: Change `ConditionallySelectable` supertrait from `Copy` to `Sized`](https://github.com/dalek-cryptography/subtle/pull/137)
- [#136: WIP breaking changes for a hypothetical v3.0](https://github.com/dalek-cryptography/subtle/pull/136)
- [#134: `const fn` support](https://github.com/dalek-cryptography/subtle/issues/134)

These are just some of the issues and PRs to `subtle` that I have opened. Unfortunately with no
reviewers these and PRs by others have gone unreviewed.

If it were possible to find reviewers for `subtle` and get it to a state where it has the major
functionality we're looking for from `ctutils`, making a small number of breaking changes to support
heap-allocated types, that would probably be the best outcome. Though we have already started
releasing RustCrypto crates that use `ctutils`, including `crypto-bigint`, if `subtle` could be
updated we could probably find a way to make it work, though as we release more and more crates it
becomes harder and harder.

# Prior art
[prior-art]: #prior-art

`subtle` is, of course, the prior art here.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TODO

# Future possibilities
[future-possibilities]: #future-possibilities

TODO
