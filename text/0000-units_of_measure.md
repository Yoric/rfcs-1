- Feature Name: units_of_measure
- Start Date: 2018-03-28
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

We propose to extend the Rust type system and standard library to let users
attach physical units (e.g. meters, meter per second), as well as other
kinds of units (e.g. dollars per galleon, children per inhabitant) to data.
We further propose a mechanism to combine maintain this information throughout
arithmetic operations (e.g. dividing meters per seconds yields meters per second).

# Motivation
[motivation]: #motivation

When observing the real world, and in particular measuring and reasoning on real world
value, everybody uses *units of measure*: the distance is 5 *meters* or 5 *inches*, the speed is
25 *miles per hour*, the duration 51.3 *seconds*, the crowd is 4,000 *people*, the median
age is 42 *years*, etc.

In programming, the equivalent concept is types. Sadly, representing
such quantified values with types is often either something that needs
to be implemented manually for each possible unit of measure and each
conversion. Although the rich type system of Rust (or Haskell, or C++)
can be tricked into representing many units of measures without too much
boilerplate **CITATIONS NEEDED**, the results remain limited,
non-compositional, and the error messages are hard to read. The main
exception is F#, which offers a units of measure
in the type system, with full inference and compositionality.

The objective of this proposal is to enrich the Rust type system and
standard library with a notion of unit of measure, supporting
type inference, extensibility, compositionality, and safe arithmetic
operations.

This proposal is very strongly inspired from F#'s unit of measure system
[*Types for units-of-measure: Theory and practice*, by Andrew Kennedy](https://scholar.google.fr/citations?hl=en&user=n48ZVvQAAAAJ&view_op=list_works&sortby=pubdate#d=gs_md_cita-d&p=&u=%2Fcitations%3Fview_op%3Dview_citation%26hl%3Den%26user%3Dn48ZVvQAAAAJ%26cstart%3D20%26pagesize%3D80%26sortby%3Dpubdate%26citation_for_view%3Dn48ZVvQAAAAJ%3A3fE2CSJIrl8C%26tzom%3D-120).
By opposition to that work, though, we do not assume that `0` is polymorphic
in unit of measure.


# Detailed design
[design]: #detailed-design

Let us start by an example of what we wish to achieve:

```rust
/// Define a unit of measure: the engineer.
struct Engineer;
impl Unit for Engineer {}

/// Define a unit of measure: the lightbulb.
struct Lightbulb;
impl Unit for Lightbulb {}

// Determine experimentally how many engineers it takes to change
// `n` lightbulbs, deduce an average number needed to change one
// lightbulb.
fn how_many_engineers_does_it_take_to_change_a_lightbulb(rng: &mut RNG, sample_size: usize) -> Measure<f64, Engineer/Lightbulb> {
    let mut total_number_of_engineers = Measure<0, Engineer>;
    for i in 0..sample_size {
        total_number_of_engineers += rng.next_u32() ;
    }
    let total_number_of_lightbulbs = Measure<sample_size, Lightbulb>;

    total_number_of_engineers.into() / total_number_of_lightbulbs.into() // Convert to f64 before division, then return.
}
```

In the above, `Engineer/Lightbulb` is syntactic sugar for a more complicated type.

Note that the example above attaches a unit to `usize`, `u32` and `f64`, but there
is no reason to limit ourselves to usual numerical types. Indeed, some applications
(e.g. banking) require using fixed-point arithmetics, while others may prefer using
rationals, big integers, big rationals, etc. Similarly, there is no reason to prevent
vectors from having units as a whole. For this reason, in the current proposal, units
can be attached to any value.

## Standard library (user-facing)

We extend the standard library with a new module `units`.

This module defines the following user-level traits and structs:

```rust
/// A unit of measure.
///
/// This trait is meant to be used mainly with void structs, but can
/// be implemented by any type.
pub trait Unit {}

/// A value with a unit.
pub struct Measure<T, U: Unit> {
    value: T,
    unit: PhantomData<U>,
}

/// A dimensionless value.
///
/// This trait constitutes a proof obligation for the type-checker.
///
/// This trait is special, insofar as the type-checker ensures
/// that it can only be implemented by the type combinators
/// defined below.
trait ODimensionless: Unit {}

/// The type-level product of two units of measure.
///
/// This trait constitutes a proof obligation for the type-checker.
///
/// This trait is special, insofar as the type-checker ensures
/// that it can only be implemented by the type combinators
/// defined below.
pub trait OMul: Unit where Self::Left: Unit, Self::Right: Unit {
    type Left;
    type Right;
}

/// The type-level product of two units of measure.
///
/// This trait constitutes a proof obligation for the type-checker.
///
/// This trait is special, insofar as the type-checker ensures
/// that it can only be implemented by the type combinators
/// defined below.
pub trait OInv: Unit where Self::Inner: Unit {
    type Inner;
}
```

We also define the common operations on `Measure`, as follows:

```rust
/// Build a Measure<T, U> from a T.
impl<T, U: Unit> From<T> for Measure<T, U> {
    // ...
}

/// Convert the container for a value, without changing the unit.
/// (e.g. convert from usize to isize).
impl<T, U: Unit, V> From<Measure<V, U>> for Measure<T, U> where T: From<V> {
    // ...
}
impl<T, U: Unit, V> Into<Measure<V, U>> for Measure<T, U> where T: Into<V> {
    // ...
}
impl<T, U: Unit, V, E> TryFrom<Measure<V, U>, Error = E> for Measure<T, U> where T: TryInto<V, Error = E> {
    // ...
}
impl<T, U: Unit, V, E> TryInto<Measure<V, U>, Error = E> for Measure<T, U> where T: TryInto<V, Error = E> {
    // ...
}

impl<T, U: Unit> AsRef<T> for Measure<T, U> {
    // ...
}
impl<T, U: Unit> Debug for Measure<T, U> where T: Debug {
    // ...
}
impl<T, U: Unit> Clone<T> for Measure<T, U> where T: Clone {
    // ...
}
impl<T, U: Unit> Copy<T> for Measure<T, U> where T: Copy { }

/// Equality
impl<T, U: Unit, V> PartialEq<T, Rhs = Measure<V, U>> for Measure<T, U> where T: PartialEq<Rhs = V> {
    // ...
}
impl<T, U: Unit> Eq<T> for Measure<T, U> where T: Eq { }

impl<T, U: Unit, V> PartialOrd<T, Rhs = Measure<V, U>> for Measure<T, U> where T: PartialOrd<Rhs = V> {
    // ...
}
impl<T, U: Unit> Ord<T> for Measure<T, U> where T: Ord { }

/// Out of the box, one may only add two values with the same unit.
///
/// It remains, however, possible to manually implement Add e.g. for
/// a time and a duration or a point and a vector.
impl<T, U: Unit> Add<Self> for Measure<T, U> where T: Add<Output = T> {
    type Output = Self;
    fn add(self, rhs: Self) -> Self {
        Measure(self.0 + rhs.0)
    }
}

/// Multiply a dimensionless value with a measure.
///
/// ```rust
/// let one_meter : Measure<isize, Meter> = Measure::from(1);
/// let ten_meters : Measure<isize, Meter> = 10 * one_meter;
/// ```
impl<T, U: Unit> std::ops::Mul<Measure<T, U>> for T where T: std::ops::Mul<T> {
    type Output = std::ops::Mul<Measure<T, U>>;
    fn add(self, rhs: Measure<T, U>) -> Self::Output {
        Measure(self * rhs.0)
    }
}

impl<T, U: Unit> std::ops::Mul<Self> for Measure<T, U> where T: std::ops::Mul<T> {
    type Output = Self;
    fn add(self, rhs: Measure<T, U>) -> Self::Output {
        Measure(self.0 * rhs.0)
    }
}

/// Multiply by a dimensioned value.
impl<T, U: Unit, V: Unit, W: Unit> Mul<Measure<T, V>> for Measure<T, U> where T: std::ops::Mul<T>,
    W: OMul<Left = U, Right = V>
{
    type Output = Measure<T, W>;
    fn add(self, rhs: Measure<T, V>) -> Self::Output {
        Measure(self.0 * rhs.0)
    }
}

// ... other arithmetic operations.
```

## Standard library (embedded logics)

We further extend the standard library with the following
types. These types are primarily meant to be injected by
the compiler after type inference, to represent the
canonical implementation of a type implementing a given
unit signature.

```rust
pub struct PDimensionless;
impl Unit for PDimensionless {}
impl ODimensionless for PDimensionless {}

/// Exposing type-level product.
pub struct PMul<A, B> where A: Unit, B: Unit {
    left: PhantomData<A>,
    right: PhantomData<B>,
}
impl<A: Unit, B: Unit> Unit for PMul<A, B> {}
impl<A: Unit, B: Unit> OMul for PMul<A, B> {
    type Left = A;
    type Right = B;
}

/// Exposing type-level inversion.
pub struct PInv<A> where A: Unit {
    inner: PhantomData<A>
}
impl<A: Unit> Unit for PInv<A> {}
impl<A: Unit> OInv for PInv<A> {
    type Inner = A;
}

/// Exposing type-level commutativity
pub struct PComm<A> where A: OMul {
    inner: PhantomData<A>
}
impl<A: OMul> Unit for PComm<A> {}
impl<A: OMul> OMul for PComm<A> {
    type Left  = A::Right;
    type Right = A::Left;
}

/// Exposing type-level associativity
pub struct PAssoc<A: Unit, B: OMul> {
    left: PhantomData<A>,
    right: PhantomData<B>,
}
impl<A: Unit, B: OMul> Unit for PAssoc<A, B> {}
impl<A: Unit, B: OMul> OMul for PAssoc<A, B> {
    type Left = PMul<A, B::Left>;
    type Right = B::Right;
}

/// Inverse
impl<A: Unit, B: OInv<Inner = A>> ODimensionless for PMul<A, B> {

}

/// Neutral element is its own inverse
impl<A: ODimensionless> ODimensionless for PInv<A> {

}


/// Exposing neutrality of Id
struct PId<A: OMul> where A::Left : ODimensionless {
    inner: PhantomData<A>,
}
impl<A: OMul> Unit for PId<A> where A::Left : ODimensionless  { }
impl<A: OMul> OMul for PId<A> where A::Left : ODimensionless, A::Right: OMul  {
    type Left = <<A as OMul>::Right as OMul>::Left;
    type Right = <<A as OMul>::Right as OMul>::Right;
}
impl<A: OMul> OInv for PId<A> where A::Left : ODimensionless, A::Right: OInv  {
    type Inner = <<A as OMul>::Right as OInv>::Inner;
}
```

Our objective, at this stage, is to find the smallest
chain of `PId`, `PMul`, ... nodes needed to produce an
element of type `ODimensionless`, `OMul<A, B>`, ...

The main complication being type variables.

**FIXME** What kind of power do we lose if we don't care about type
variables in this RFC?

## Type system

### Traits

The traits `OMul`, `OInv`, `ODimensionless` may only be implemented
by the structs above.

**FIXME** What's the worst that can happen if someone implements them?

### Unit expressions

(pretty much everything from this point is ported from Andrew Kennedy's work)

We gather all the `where U: PMul<V, W>`, `where U: PInv<V>` and `where U: ODimensionless`
constraints as a set of unification obligations expressed as *Unify(u, v)*, where *u* and *v*
are defined using the following grammar for unit expressions:

```
u, v ::= 1      // Dimensionless
      |  u * v  // A constraint along the lines of TMul<U, V>
      |  u^-1   // A constraint along the lines of TInv<U>
      |  s      // The name of a concrete type implementing `Unit`
      |  α      // A type variable

```

As a shorthand, we write `u^2` (respectively `u^3`, etc.) for `u*u` (respectively `u*u*u`, etc.)
and `u^-2` (respectively `u^-3`, etc.) for `u^-1 * u^-2` (respectively `u^-1 * u^-1 * u^-1`, etc.)

Note that `s` represent only base types implementing `Unit`, not type aliases.


### Equational theory

The type system implements the following equational theory (ported from **CITATION NEEDED**)

```
------ (REFL)
u == u
```


```
u == v
------ (SYM)
v == u
```


```
u == v   v == w
--------------- (TRANS)
u == w
```


```
u == v
------------ (CONG1)
u^-1 == v^-1
```

```
u == v    u2 == v2
------------------ (CONG2)
u * v == u2 * v2
```

```
---------- (ID)
u * 1 == u
```

```
----------------------------- (ASSOC)
u * ( v * w ) == ( u * v) * w
```

```
-------------- (COMM)
u * v == v * u
```

```
-------------- (INV)
u * u ^-1 == 1
```

### Normal form

This equational theory gives us the ability to introduce a canonical representation
for a unit expression: `α_1 ^ n_1 * ... * α_p ^ n_p * s_1 ^ m_1 * ... s_q ^ n_q * 1`
where `α_1`, ... `α_p` are distinct and ranked by lexicographical order and
`s_1`, ... `s_q` are distinct and ranked by lexicographical order.



### Unification

We first note that attempting to solve *Unify(u, v)* is the same as attempting
to solve *Unify(u * v^-1, 1)*.

We then use the following algorithm, in pseudo-Rust:

```rust
fn unify_with_one(expr: &UnitExpression) {
    let expr = expr.normalized();
    if expr.type_variables().len() == 0 {
        // We have nothing to substitute.

        // We do not count `1` in `base_units`, despite the fact that it is materialized
        // as concrete type `Dimensionless`.
        if expr.base_units().len() == 0 {
            // No base units, either, so the equation is actually 1 = 1
            return Ok(Substitution::identity())
        }
        // Otherwise, the equation is non solvable.
        return Err(Error::NoSolution);
    }

    if expr.type_variables().len() == 1 {
        let degree = expr.type_variables()[0].degree;
        if expr.base_units()
            .iter()
            .all(|base| base.degree mod degree == 0)
        {
            // Simple solution.
            let solution = expr.base_units()
                .drain()
                .map(|base| Base {
                    name: base.name,
                    degree: -base.degree / degree
                });
            return Ok(Substitution::from(solution))
        }
        // Otherwise, the equation is non solvable.
        return Err(Error::NoSolution);
    }


    // Find the type variable with the smallest degree, we'll use it to
    // simplify the equation.
    let min = expr.type_variables()
        .iter()
        .enumerate()
        .min_by(|(_, v1), (_, v2)| Ord::cmp(v1.degree, v2.degree))
        .unwrap(); // We know that the list is non-empty.

    let mut substitution_0 = UnitExpression::new();
    for (i, var) in expr.type_variables().iter().enumerate() {
        if i == min.0 {
            continue;
        }
        substitution_0 *= TypeVar {
            name: var.name.clone(),
            degree: - (var.degree / min.degree)
        }
    }
    for base in expr.base_units().iter() {
        substitution_0 *= Base {
            name: base.name.clone(),
            degree: - (base.degree / min.degree)
        }
    }
    let simplification = Substitution::new(
        min.name.clone(),
        substitution_0.clone()
    );

    // This substitution effectively applies a `mod x_i` to the exponent of all
    // type variables and base units.
    let substitution = unify_with_one(expr.substitute(simplification))?;
    Ok(substitution * simplification)
}
```

See A.J. Kennedy *Programming Languages and Dimensions*, PhD thesis, for a proof
that this is the most general unifier.

### Canonical types

This unification gives us the ability to introduce a canonical implementation
of `PInv<U>` and `PMul<U, V>` constraints by converting each substitution
to a chain of `PMul`, `PInv`, `PComm`, ...

**FIXME** At least, I think so. Detail how.


## Syntactic sugar

When parsing a type expression:

- `U * V` is the name of the canonical implementation of `PMul<U, V>`;
- `U^-1` is the name of the canonical implementation `PInv<U>`.
- `U / V` is the same thing as `U * V^-1`;

When printing an error message:

- `U * V` is used to represent the constraint `PMul<U, V>`;
- `U^-1` is used to represent the constraint `PInv<U>`.



# Drawbacks
[drawbacks]: #drawbacks

This complicates the type system.

There may be a way to split several concerns and land them separately.

# Alternatives
[alternatives]: #alternatives

Several other approaches have been proposed to offer units of measure in Rust:

- [uom](https://crates.io/crates/uom);
- [dimensioned](https://crates.io/crates/dimensioned);
- [rink](https://crates.io/crates/rink).

These approaches are all based on crates and macros. They
work when the comprehensive set of units is known in advance,
but cannot let units from two distinct libraries interact. Also, due to
type-level integer arithmetics, error messages are very complex.

The C++ Boost library also offers a [user-level implementation of units of measure](http://www.boost.org/doc/libs/1_66_0/doc/html/boost_units/Quick_Start.html),
also based on type-level integer arithmetics. While this approach is a bit
more extensible, it relies upon magic numbers whose existence must be kept
distinct between libraries by the programmer to avoid collisions, and
it suffers from the usual Boost problem of very complex error messages.


# Unresolved questions
[unresolved]: #unresolved-questions

We have not attempted to provide the most generic definition possible of
`Add`, `Mul`, etc. We are content with simpler definitions, until a possible
followup RFC.

What parts of the design are still TBD?
