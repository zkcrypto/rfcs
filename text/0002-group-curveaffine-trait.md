- Feature Name: `group_curveaffine_trait`
- Start Date: 2026-02-26
- RFC PR: [zkcrypto/rfcs#0002](https://github.com/zkcrypto/rfcs/pull/0002)
- Tracking Issue: [zkcrypto/rfcs#0000](https://github.com/zkcrypto/rfcs/issues/0000)

# Summary
[summary]: #summary

Unify the methods previously exposed by the `PrimeCurveAffine` and
`CofactorCurveAffine` traits. The prime-order and cofactor traits all become
marker traits, and their affine-specific traits are automatically derived.

# Motivation
[motivation]: #motivation

Currently it is not possible to access the affine-form curve element APIs
without first constraining to either `PrimeCurveAffine` or
`CofactorCurveAffine`. These two traits have identical APIs, and by moving them
to a common parent trait `CurveAffine`, users can write more code that works
with either kind of curve.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When working with affine-form curve elements, you now import `CurveAffine` to
access their operations, instead of `PrimeCurveAffine` or `CofactorCurveAffine`.

`PrimeCurveAffine` is still the usual trait bound applied to generic functions,
as most cryptographic protocols require prime-order groups. However, now you can
instead use `CurveAffine` as a bound when writing generic functions that do not
depend on the curve elements being in the prime-order subgroup.

**For trait implementors:** The `PrimeCurveAffine` or `CofactorCurveAffine`
trait you previously implemented is now automatically derived. To migrate, alter
your existing trait implementations to instead implement `CurveAffine`. For
example, in the `bls12_381` crate the migration for `G1Affine` is:
```diff
 impl PrimeGroup for G1Projective {}

 impl Curve for G1Projective {
-    type AffineRepr = G1Affine;
+    type Affine = G1Affine;

-    fn batch_normalize(p: &[Self], q: &mut [Self::AffineRepr]) {
+    fn batch_normalize(p: &[Self], q: &mut [Self::Affine]) {
         Self::batch_normalize(p, q);
     }

-    fn to_affine(&self) -> Self::AffineRepr {
+    fn to_affine(&self) -> Self::Affine {
         self.into()
     }
 }

-impl PrimeCurve for G1Projective {
-    type Affine = G1Affine;
-}
+impl PrimeCurve for G1Projective {}

-impl PrimeCurveAffine for G1Affine {
+impl CurveAffine for G1Affine {
     type Scalar = Scalar;
     type Curve = G1Projective;
```

For prime-order curves, if you previously implemented `CofactorCurveAffine` as
well in order to use code written for that trait (via shell calls to the
`PrimeCurveAffine` impl), you no longer need it; a single `CurveAffine` impl
suffices.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The following new trait is introduced:
```rust
/// Affine representation of an elliptic curve point.
pub trait CurveAffine:
    GroupEncoding
    + Copy
    + fmt::Debug
    + Eq
    + Send
    + Sync
    + 'static
    + Neg<Output = Self>
    + Mul<<Self::Curve as Group>::Scalar, Output = Self::Curve>
    + for<'r> Mul<&'r <Self::Curve as Group>::Scalar, Output = Self::Curve>
{
    /// The efficient representation for this elliptic curve.
    type Curve: Curve<Affine = Self, Scalar = Self::Scalar>;

    /// Scalars modulo the order of this group's scalar field.
    ///
    /// This associated type is temporary, and will be removed once downstream users have
    /// migrated to using `Curve` as the primary generic bound.
    type Scalar: PrimeField;

    /// Returns the additive identity.
    fn identity() -> Self;

    /// Returns a fixed generator of unknown exponent.
    fn generator() -> Self;

    /// Determines if this point represents the additive identity.
    fn is_identity(&self) -> Choice;

    /// Converts this affine point to its efficient representation.
    fn to_curve(&self) -> Self::Curve;
}
```

The `Curve::AffineRepr` associated type is replaced with a `Curve::Affine`
associated type bounded on `CurveAffine`:
```rust
// BEFORE:
pub trait Curve:
    Group + GroupOps<<Self as Curve>::AffineRepr> + GroupOpsOwned<<Self as Curve>::AffineRepr>
{
    type AffineRepr;

    fn batch_normalize(p: &[Self], q: &mut [Self::AffineRepr]) {
        // ...
    }
    fn to_affine(&self) -> Self::AffineRepr;
}

// AFTER:
pub trait Curve: Group + GroupOps<Self::Affine> + GroupOpsOwned<Self::Affine> {
    type Affine: CurveAffine<Curve = Self, Scalar = Self::Scalar>
        + Mul<Self::Scalar, Output = Self>
        + for<'r> Mul<&'r Self::Scalar, Output = Self>;

    fn batch_normalize(p: &[Self], q: &mut [Self::Affine]) {
        // ...
    }
    fn to_affine(&self) -> Self::Affine;
}
```

The `PrimeCurve::Affine` and `CofactorCurve::Affine` associated types are removed
(replaced by `Curve::Affine`):
```rust
// BEFORE:
pub trait PrimeCurve: Curve<AffineRepr = <Self as PrimeCurve>::Affine> + PrimeGroup {
    type Affine: PrimeCurveAffine<Curve = Self, Scalar = Self::Scalar>
        + Mul<Self::Scalar, Output = Self>
        + for<'r> Mul<&'r Self::Scalar, Output = Self>;
}
pub trait CofactorCurve:
    Curve<AffineRepr = <Self as CofactorCurve>::Affine> + CofactorGroup
{
    type Affine: CofactorCurveAffine<Curve = Self, Scalar = Self::Scalar>
        + Mul<Self::Scalar, Output = Self>
        + for<'r> Mul<&'r Self::Scalar, Output = Self>;
}

// AFTER:
pub trait PrimeCurve: Curve + PrimeGroup {}
pub trait CofactorCurve: Curve + CofactorGroup {}
```

The `PrimeCurveAffine` and `CofactorCurveAffine` traits have their contents removed,
becoming marker traits with a bound on `CurveAffine` and automatic derivation:
```rust
// BEFORE:
pub trait PrimeCurveAffine: /* ... */ {
    type Scalar: PrimeField;
    type Curve: PrimeCurve<Affine = Self, Scalar = Self::Scalar>;
    // ...
}
pub trait CofactorCurveAffine: /* ... */ {
    type Scalar: PrimeField;
    type Curve: CofactorCurve<Affine = Self, Scalar = Self::Scalar>;
    // ...
}

// AFTER:
pub trait PrimeCurveAffine: CurveAffine {}
pub trait CofactorCurveAffine: CurveAffine {}
impl<C: CurveAffine> PrimeCurveAffine for C where C::Curve: PrimeCurve {}
impl<C: CurveAffine> CofactorCurveAffine for C where C::Curve: CofactorCurve {}
```

# Drawbacks
[drawbacks]: #drawbacks

Every crate that implements `group` traits will need to update their
implementations. However, the migration is mechanical and should be straightforward.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- The `CurveAffine::Scalar` associated type is included solely to help
  downstream users that built their internal trait hierarchies around
  `C: PrimeCurveAffine` instead of `C: PrimeCurve`.
- We could continue to not make this change, but that would suck.

# Prior art
[prior-art]: #prior-art

The trio of `Group`, `PrimeGroup`, and `CofactorGroup` traits have had this same
kind of relationship for a long time, with the common methods on `Group`, and
`PrimeGroup` being just a marker trait.

There has been a similar relationship between the `Curve`, `PrimeCurve`, and
`CofactorCurve` traits, which lived on top of the `Group`-related traits.

The only reason that `CurveAffine` didn't exist earlier, is that this trio lives
*beside* the `Curve`-related traits via associated types, and the Rust compiler
couldn't handle the resulting trait resolution problem.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None. `CurveAffine` has been proposed for years, and no one has expressed issues
with it.

# Future possibilities
[future-possibilities]: #future-possibilities

At some point, `CurveAffine::Scalar` should be removed, and users should instead
build their APIs around the `Curve` trait. However, the main downstream that
would be affected here is the `pasta_curves` crate, which cannot migrate to
`Curve` until we have trait APIs for handling curve coordinates (so that its
`CurveExt` extension trait can be removed).
