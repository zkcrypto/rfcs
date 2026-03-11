- Feature Name: `rand_core_0_10`
- Start Date: 2026-02-22
- RFC PR: [zkcrypto/rfcs#1](https://github.com/zkcrypto/rfcs/pull/1)
- Tracking Issue: [zkcrypto/rfcs#0000](https://github.com/zkcrypto/rfcs/issues/0000)

# Summary
[summary]: #summary

Upgrade the `ff` and `group` crates (and their downstream crates, e.g. `pairing`, `bls12_381`, and `jubjub`) from `rand_core 0.6` to `rand_core 0.10`, updating all public trait APIs to reflect the renamed and restructured traits in the new `rand_core` release. This is a semver-breaking change requiring new major releases of the affected crates.

# Motivation
[motivation]: #motivation

`rand_core 0.10` was released on February 1st, 2026, with significant trait renames and API restructuring, described by its maintainers as the ["last significant breakage before 1.0."](https://github.com/rust-random/rand_core/releases/tag/v0.10.0)

The zkcrypto trait crates (`ff`, `group`) expose `rand_core` types in their public APIs, specifically in `Field::random` and `Group::random`. Because `rand_core` is a public dependency, any upgrade to a new semver-incompatible version of `rand_core` requires a corresponding semver-breaking release of the zkcrypto crates.

There is strong motivation to make this upgrade now.

### Ecosystem alignment

The RustCrypto project (which maintains trait crates like `elliptic-curve` and `digest`) has decided to [skip `rand_core 0.9` entirely](https://github.com/zkcrypto/ff/pull/130#issuecomment-3565178513) and move directly to `rand_core 0.10`. End users that combine zkcrypto and RustCrypto crates in the same dependency tree would otherwise be forced to maintain incompatible duplicates of the `rand_core` traits.

### Avoiding double disruption

The zkcrypto crates currently depend on `rand_core 0.6`. We originally planned an intermediate upgrade to `rand_core 0.9`, but since the broader ecosystem (including RustCrypto) is skipping 0.9, performing a single jump from 0.6 to 0.10 is less disruptive to downstream users than two sequential breaking upgrades.

### Downstream demand

Multiple contributors and downstream projects have requested this upgrade[^demand], and prototype work has already been done on the `release-0.14.0` branches of `ff` and `group`.

### MSRV is reasonable

Rust 1.85 (the Rust 2024 edition boundary) was released over a year ago, and is already the MSRV for both the RustCrypto crates and popular `ff` consumers like [`librustzcash`](https://github.com/zcash/librustzcash).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## MSRV bump

The minimum supported Rust version for all affected crates is raised to **1.85.0**. This is required by `rand_core 0.10`.

## `Field::random` and `Group::random`

The `random` method signature changes from taking ownership of an RNG to taking a mutable reference:

```rust
// BEFORE:
fn random(rng: impl RngCore) -> Self;

// AFTER:
fn random<R: Rng + ?Sized>(rng: &mut R) -> Self;
```

The bound changes from `RngCore` to `Rng` (which is the new name for the same concept in `rand_core 0.10`). The method now also takes `&mut R` instead of `impl RngCore`, and relaxes the `Sized` requirement on the RNG.

**Migration:** Most call sites that pass `&mut rng` will continue to work. Call sites that pass an RNG by value (e.g. `Field::random(OsRng)`) will need to change to pass by reference (e.g. `Field::random(&mut OsRng)`). Since `OsRng` has been removed from `rand_core`, users should migrate to `getrandom::SysRng` (from the [`getrandom`](https://crates.io/crates/getrandom) crate) or equivalent.

## `Field::try_random` and `Group::try_random` (new)

A new fallible method is introduced for random element generation:

```rust
fn try_random<R: TryRng + ?Sized>(rng: &mut R) -> Result<Self, R::Error>;
```

This is now the *required* method of the trait; `random` becomes a *provided* method that delegates to `try_random` and unwraps the infallible result. This enables use of fallible RNG sources (e.g. hardware RNGs that can fail) without panicking.

**For trait implementors:** You now implement `try_random` instead of `random`. The `random` method has a default implementation that calls `try_random` and handles the `Infallible` error case. Note that the bound on `try_random` uses `TryRng` (renamed from `TryRngCore` in `rand_core 0.10`). Inside your implementation, RNG method calls also change to their fallible counterparts (e.g. `rng.next_u64()` becomes `rng.try_next_u64()?`).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## `ff` crate changes

### Dependency changes

| Dependency | Before | After |
|---|---|---|
| `rand_core` | `0.6` | `0.10` |
| MSRV | `1.56` | `1.85` |

### `Field` trait changes

The `Field` trait's RNG-related methods change as follows:

```rust
// BEFORE:
use rand_core::RngCore;

pub trait Field: /* ... */ {
    fn random(rng: impl RngCore) -> Self;
    // ...
}

// AFTER:
use rand_core::{Rng, TryRng};

pub trait Field: /* ... */ {
    /// Provided method (infallible RNG).
    fn random<R: Rng + ?Sized>(rng: &mut R) -> Self {
        // `Rng: TryRng<Error = Infallible>`, so this is exhaustive.
        let Ok(out) = Self::try_random(rng);
        out
    }

    /// Required method (fallible RNG).
    fn try_random<R: TryRng + ?Sized>(rng: &mut R) -> Result<Self, R::Error>;
    // ...
}
```

Key details:
- `random` changes from a **required** method to a **provided** method.
- `try_random` is the new **required** method that trait implementors must provide.
- The `?Sized` bound on `R` allows passing trait objects (e.g., `&mut dyn Rng`), which was not possible before.
- `Rng` is an extension trait of `TryRng<Error = Infallible>`, so any infallible RNG satisfies the `TryRng` bound used by `try_random`.

## `group` crate changes

### Dependency changes

| Dependency | Before | After |
|---|---|---|
| `ff` | `0.13` | next major |
| `rand_core` | `0.6` | `0.10` |
| MSRV | `1.56` | `1.85` |

### `Group` trait changes

Mirrors the `Field` trait changes:

```rust
// BEFORE:
pub trait Group: /* ... */ {
    fn random(rng: impl RngCore) -> Self;
    // ...
}

// AFTER:
pub trait Group: /* ... */ {
    fn random<R: Rng + ?Sized>(rng: &mut R) -> Self { /* default impl */ }
    fn try_random<R: TryRng + ?Sized>(rng: &mut R) -> Result<Self, R::Error>;
    // ...
}
```

## Downstream crate impact

Downstream crates that implement `Field` or `Group` will need mechanical updates: each `random` implementation becomes a `try_random` implementation, and call sites are updated to use the new signatures. Crates that transitively depend on `ff` or `group` without directly using `rand_core` types (e.g. `pairing`) only need a version bump. The zkcrypto-maintained downstream crates (`bls12_381`, `jubjub`, `pairing`) will receive corresponding updates as part of this effort.[^bls12-prototype]

## Trait rename mapping (`rand_core 0.6` → `rand_core 0.10`)

For reference, the full trait rename mapping relevant to zkcrypto:

| `rand_core 0.6` | `rand_core 0.10` |
|---|---|
| `RngCore` | `Rng` |
| N/A | `TryRng` |
| `CryptoRng` | `CryptoRng` |
| N/A | `TryCryptoRng` |

Note: The `release-0.14.0` branches originally targeted `rand_core 0.9`; this RFC proposes targeting `rand_core 0.10` with the updated trait names.

# Drawbacks
[drawbacks]: #drawbacks

- **MSRV jump from 1.56 to 1.85.** This is a significant MSRV increase (29 versions). However, `rand_core 0.10` hard-requires 1.85 (it uses the Rust 2024 edition), so there is no way to avoid this bump while upgrading. In practice, the primary downstream consumers (Zcash ecosystem, RustCrypto) already require 1.85.

- **Breaking change for all downstream crates.** Every crate that implements `Field` or `Group` will need to update their implementations. However, the migration is mechanical (rename `random` to `try_random`, update the signature), and the old `random` method continues to exist as a provided method with the same semantics.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why skip `rand_core 0.9`?

We originally planned to upgrade to `rand_core 0.9` (see the `release-0.14.0` branches). However:

- RustCrypto decided to [skip `rand_core 0.9` entirely](https://github.com/zkcrypto/ff/pull/130#issuecomment-3565178513) and move to `rand_core 0.10`.
- Releasing `ff` with `rand_core 0.9` would be ["nigh-unusable"](https://github.com/zkcrypto/ff/pull/130#issuecomment-3565178513) for downstream crates that also depend on RustCrypto trait crates, since those would require `rand_core 0.10`.
- Performing a single breaking change (0.6 → 0.10) causes less ecosystem churn than two sequential breaking changes (0.6 → 0.9 → 0.10).

## Why make `try_random` the required method instead of `random`?

The `rand_core 0.9`/`0.10` design philosophy makes fallibility the primary interface, with infallible usage as a convenience wrapper. By making `try_random` the required method:

- Implementors only need to write one implementation that works for both fallible and infallible RNGs.
- The `random` method "just works" via the default implementation.
- This aligns with the direction of `rand_core` itself and addresses [ff#109](https://github.com/zkcrypto/ff/issues/109).

## Why not wait for `rand_core` 1.0?

`rand_core 0.10` is explicitly described as the "last significant breakage before 1.0." Waiting for 1.0 would leave the zkcrypto ecosystem stuck on `rand_core 0.6` indefinitely while the rest of the Rust ecosystem moves forward. Additionally, the transition from 0.10 to 1.0 should be minimally disruptive.

## What happens if we don't do this?

Not upgrading would leave the zkcrypto ecosystem unable to interoperate with the post-`rand_core 0.10` versions of RustCrypto crates, effectively forking the Rust cryptography ecosystem into incompatible dependency trees. This would be particularly problematic for downstream crates (e.g. `bls12_381`) that bridge both ecosystems.

# Prior art
[prior-art]: #prior-art

- **RustCrypto ecosystem.** The RustCrypto project is performing the same `rand_core 0.10` migration across all of their trait and implementation crates. Their decision to skip `rand_core 0.9` directly informed our approach. See [RustCrypto/meta#19](https://github.com/RustCrypto/meta/issues/19) for their discussion of RNG API conventions, including fallible RNG interfaces.

- **`rand_core` 0.10 release.** The [release notes](https://github.com/rust-random/rand_core/releases/tag/v0.10.0) document the rationale for the trait renames (`RngCore` → `Rng`, `TryRngCore` → `TryRng`) and the removal of OS RNG functionality to the `getrandom` crate.

- **`getrandom` 0.4.** The `getrandom` crate has taken over the OS-level RNG functionality previously in `rand_core` (specifically `OsRng`), now exposed as `getrandom::SysRng` (introduced in `getrandom 0.4`).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **Release version numbers for `ff` and `group`.** Both crates are expected to release as version **0.14**, but the exact version numbers will be confirmed during implementation.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Removing the `random` convenience method.** In the long term, it may make sense to deprecate or remove `Field::random` / `Group::random` in favor of exclusively using `try_random`, aligning fully with the `rand_core` design philosophy. However, this is not proposed in this RFC.

- **Edition 2024 migration.** With MSRV 1.85, all crates could migrate to `edition = "2024"` in a follow-up release.

[^demand]: [ff#109](https://github.com/zkcrypto/ff/issues/109), [ff#122](https://github.com/zkcrypto/ff/pull/122), [ff#147](https://github.com/zkcrypto/ff/pull/147), [group#56](https://github.com/zkcrypto/group/pull/56)
[^bls12-prototype]: A prototype migration PR for `bls12_381` already exists: [bls12_381#145](https://github.com/zkcrypto/bls12_381/pull/145).