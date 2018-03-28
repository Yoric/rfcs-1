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
exception is F# **CITATION NEEDED**, which offers a units of measure
in the type system, with full inference and compositionality.

The objective of this proposal is to enrich the Rust type system and
standard library with a notion of unit of measure, supporting
type inference, extensibility, compositionality, and safe arithmetic
operations.

This proposal is strongly inspired from F#'s unit of measure system
**CITATION NEEDED**.


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

## Standard library

We extend the standard library with a new module `units`.

This module defines the following traits and structs:

```rust
/// A unit of measure.
///
/// This trait is special, insofar as the type-checker ensures
/// that it is only applied to either void types or the type
/// combinators defined below.
///
/// Examples:
/// ```
/// pub struct Meter;
/// impl Unit for Meter {};
/// ```
pub trait Unit {}

/// A value with a unit.
pub struct Measure<T, U: Unit> {
    value: T,
    unit: PhantomData<U>
};

/// A dimensionless value.
pub struct Dimensionless;
impl Unit for Dimensionless {}

/// The type-level product of two units of measure.
pub trait TMul<A, B> where A: Unit, B: Unit {}
impl<A: Unit, B: Unit> Unit for TMul<A, B> {}

/// The type-level inverse of a unit of measure.
pub trait TInv<A> where A: Unit {}
impl<A: Unit, B: Unit> Unit for TInv<A, B> {}
```

Note that `Unit`, `TMul` and `TInv` are special traits, used internally by the compiler. More on this
in the next section.


Furthermore, common operations are defined for `Measure`, as follows:

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

/// During multiplication,
impl<T, U: Unit, V: Unit, W: Unit> Mul<Measure<T, V>> for Measure<T, U> where T: Mul<T>, W: TMul<U, V> {
    type Output = Measure<T, W>;
    fn add(self, rhs: Measure<T, V>) ->  Self::Output {
        Measure(self.0 + rhs.0)
    }
}

// ... other arithmetic operations.
```

## Type system

We first introduce the following constraints on trait `Unit`:

- void structs may implement `Unit`;
-



## Syntactic sugar



# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?
