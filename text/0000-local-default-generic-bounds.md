- Feature Name: `local_default_bounds`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

<!-- todo: Replace with RFC PR later -->
[`forget_marker_trait`]: https://github.com/Ddystopia/rfcs/blob/leak-marker-trait-and-local-default-bounds/text/0000-forget-marker-trait.md

# Summary
[summary]: #summary

This RFC proposes a mechanism for crates to define default bounds on generics, traits, and associated types. By specifying these defaults at the crate level we can reduce the need for verbose and repetitive `?Trait` annotations while maintaining backward compatibility and enabling future language evolution.

# Motivation
[motivation]: #motivation

## What are `?Trait` bounds

Generic parameters on functions (`fn foo<T>()`), associated types in traits `trait Foo { type Assoc; }` and `Self` in traits `trait Foo where Self ...` can have `where` bounds.

This function expects any `T` that can be compared via `==` operator:

```rust
fn foo<T: PartialEq>(t: &T) {}
```

But Rust introduces some bounds by default. In the code above, `T` must be both `PartialEq` and `Sized`. To opt out of this, users need to write `+ ?Sized` manually:

```rust
fn foo<T: PartialEq + ?Sized>(t: &T) {}
```

## Use of `?Trait` bounds for new features
[applicability-of-default-bounds]: #applicability-of-default-bounds

A lot of new features (see #use-cases) require breaking old code by removing long-established assumptions like `size = stripe` or the ability to skip the destructor of a type. To avoid breaking the code, they create a new trait representing an assumption and then define their feature as types that do not implement this trait. Here `?Trait` bounds come in - old code has old assumptions, but new code can add `?Trait` to opt out of them and support more types.

It is also important to note that in most cases those assumptions are not actually exercised by generic code, they are just already present in signatures - rarely code needs `size = stride`, or to skip the destructor (especially for a foreign type).

## The problem
[problem-of-default-bounds]: #problem-of-default-bounds

Quotes from "Size != Stripe" [Pre-RFC thread](https://internals.rust-lang.org/t/pre-rfc-allow-array-stride-size/17933):

> In order to be backwards compatible, this change requires a new implicit trait bound, applied everywhere. However, that makes this change substantially less useful. If that became the way things worked forever, then `#[repr(compact)]` types would be very difficult to use, as almost no generic functions would accept them. Very few functions actually need `AlignSized`, but every generic function would get it implicitly.


@scottmdcm

> Note that every time this has come up -- `?Move`, `?Pinned`, etc -- the answer has been **"we're not adding more of these"**.
> 
> What would an alternative look like that doesn't have the implicit trait bound?

In general, many abstractions can work with both `Trait` and `!Trait` types, and only a few actually require `Trait`. For example, `Forget` bound is necessary for only a few functions in std, such as `forget` and `Box::leak`, while `Option` can work with `!Forget` types too.
However, if Rust were to introduce `?Forget`, every generic parameter in `std` would need an explicit `?Forget` bound. This would create excessive verbosity and does not scale well.

There is a more fundamental problem noted by @bjorn3: `std` would still need to have `Forget` bounds on all associated items of traits to maintain backward compatibility, as some code may depend on them. This makes `!Forget` types significantly harder to use and reduces their practicality. Fortunately, @Nadrieril proposed a solution to that problem, which resulted in that RFC.

See #guide-level-explanation for details.

## Use cases
[use-cases]: #use-cases

- `!Forget` types - types with a guarantee that destructors will run at the end of their lifetime. Those types are crucial for async and other language features, which are described in [`forget_marker_trait`] Pre-RFC. <!--  Change to RCF and update link -->
- `Size != Stride` is a [frequently requested feature](freaquently-requested-features-size-neq-stride), but it is [fundamentally backward-incompatible change that requires `?AlignSized` bound](size-neq-stride-backward-incompatibe).
- [`Must move`] types will benefit from this too, further improving async ergonomics.
- (Pre-RFC only) Feel free to suggest more use cases 😊

[freaquently-requested-features-size-neq-stride]: https://github.com/rust-lang/lang-team/blob/master/src/frequently-requested-changes.md#size--stride
[size-neq-stride-backward-incompatibe]: https://internals.rust-lang.org/t/pre-rfc-allow-array-stride-size/17933#the-alignsized-trait-and-stdarrayfrom_ref-8
[`Must move`]: https://smallcultfollowing.com/babysteps/blog/2023/03/16/must-move-types/#so-how-would-must-move-work

The expected outcome is an open road for new language features to enter the language in a backward-compatible way and allow users and libraries to adapt gradually.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The syntax is to be bikeshedded, initially, it might be with a crate-level attributes.

```rust
#![default_generic_bounds(Sized, ?Forget, PartialEq)]
#![default_trait_bounds(?Sized, ?Forget, PartialEq)]
#![default_assoc_bounds(Sized, ?Forget, PartialEq)]
#![default_foreign_assoc_bounds(?Sized, ?Forget, PartialEq)]
```

The following example demonstrates how the compiler will understand the code. (`PartialEq` is just for an illustration. Probably nobody would ever need to use this with `PartialEq`).

```rust
#![default_generic_bounds(Sized, ?Forget, PartialEq)]
#![default_trait_bounds(?Sized, ?Forget, PartialEq)]
#![default_assoc_bounds(Sized, ?Forget, PartialEq)]
#![default_foreign_assoc_bounds(?Sized, ?Forget, PartialEq)]

use std::ops::Deref;

trait Trait: Deref + ?PartialEq {
    type Assoc: Forget;
}

struct Qux;
struct Foo<T>(T);
struct Bar<T: ?PartialEq>(T);
struct Baz<T: Trait>(T, T::Target, T::Assoc);

impl Trait for &i32 {
    type Assoc = &'static str;
}

fn main() {
    let foo = Foo(Qux); //~ error[E0277]: the trait bound `Qux: PartialEq` is not satisfied
    let bar = Bar(Qux); // compiles as expected
    let baz = Baz(&3, 3, "assoc"); // compiles as expected
}
```

Code above will desugar into this:

```rust
use std::ops::Deref;

trait Trait: Deref<Target: ?Sized + ?Forget + PartialEq> + ?PartialEq + ?Sized + ?Forget
{
    type Assoc: Sized + Forget + PartialEq;
}

struct Qux;
struct Foo<T: Sized + ?Forget + PartialEq>(T);
struct Bar<T: Sized + ?Forget + ?PartialEq>(T);
struct Baz<'a, T>(T, &'a T::Target, T::Assoc)
where
    T: Sized + ?Forget + PartialEq,
    T: Trait<Target: ?Sized + ?Forget + PartialEq, Assoc: Sized + Forget + PartialEq>
;

impl Trait for &i32 {
    type Assoc = &'static str;
}

fn main() {
    let foo = Foo(Qux);
    let bar = Bar(Qux);
    let baz = Baz(&3, 3, "assoc");
}
```

Introducing this feature is backward compatible and does not require an edition.

RFC tries to be consistent with already existing handling of `Sized`.

## Example: Migrating to `Forget`

With this RFC, transitioning to `Forget` is straightforward for any `#![forbid(unsafe)]` crate:

1. Set the appropriate bounds:

```rust
#![default_generic_bounds(?Forget)]
#![default_trait_bounds(?Forget)]
#![default_assoc_bounds(?Forget)]
#![default_foreign_assoc_bounds(?Forget)]
// or, equivalently, we can highlight the default choise of `Sized` behavior:
#![default_generic_bounds(Sized, ?Forget)]
#![default_trait_bounds(?Sized, ?Forget)]
#![default_assoc_bounds(Sized, ?Forget)]
#![default_foreign_assoc_bounds(?Sized, ?Forget)]
```

2. Resolve any compilation errors by explicitly adding `+ Forget` where needed.

3. Optionally: Recurse into your dependencies, applying the same changes as needed.

Crates using `unsafe` code should beware of `ptr::write` and other unsafe ways of skipping destructors.

## The Spirit of RFC: Why we need all four (generics, `Self` in traits and associated items)

For generics it is simple enough: to avoid manually writing (multiple) `?Trait` bound *everywhere*, polluting the codebase with information that is not important, and generating a lot of boilerplate, resulting in the source gaining tens of kilobytes.

For associated types and `Self` in traits the reason and solution are more subtle: even if we change generics in all functions in `std` to get `?Trait`, old code may rely on associated types implementing `Trait`, so we can't simply make them `?Trait`.

We will not only set `?Trait` bound for associated types, but we will also desugar old code to have where clause restricting all foreign associated types and `Self` in traits to `Trait`. New code will add that trait to its defaults, easily opting in for that change (or manually writing `?Trait`).

As this is in a Pre-RFC phase I invite everyone to see how the letter is deviating from the spirit and propose fixes 😊.

## Implications on the libraries

### Relax generic bound on public API

For migrated users it is equivalent to semver's `minor` change, while not migrated uses will observe it as `patch` change.

### Weakening associated type bound or `Self` bound in trait

If user manually migrated, used a library that did not yet migrated and started relying on `T: Trait` bound, this would be breaking to them.

In case of `std` 1 and 2 are mutually exclusive (rustc is 1:1 mapped with std), so no breaking is possible. But for regular crates the issue still remains, while only for the small fraction of the users, that are actively maintaining their code (unmaintained crates are not migrating).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Introduce four trait level attibutes: `default_generic_bounds`, `default_trait_bounds`, `default_assoc_bounds`, `default_foreign_assoc_bounds`, used to (non-exhaustively) enumerate overwrides of defaults for different types of bounds.

Every trait would initally have its unique default. In practice, bounds for all traits that are stable at the date of RFC except `Sized` would default to `?Trait`. In case of `Sized`, `default_generic_bound` and `default_assoc_bound` would be `Sized`, while `default_trait_bound`[^trait-not-sized-by-default] and `default_foreign_assoc_bound` would be `?Sized`. For new "breaking" traits, all four defaults would be `Trait`.

[^trait-not-sized-by-default]: https://rust-lang.github.io/rfcs/0546-Self-not-sized-by-default.html

We can't require the list to be exhaustive, as it would not scale with new "breaking" traits.

## Desugaring

### `default_generic_bounds`

Applied for generic parameters.

```rust

fn foo<T: PartialEq + Sized>() {}
struct Bar<T: PartialEq + Sized>(T);
trait Baz<T: PartialEq + Sized> {
    type Qux<U: PartialEq + Sized>;
}

```

### `default_trait_bounds`

Applied for `Self` in traits.

```rust
trait Trait: PartialEq + ?Sized {}
```

###  `default_assoc_bounds`

Applied for declarations of associated types in traits.

```rust
trait Trait {
    type Assoc: PartialEq + Sized;
}
```

### `default_foreign_assoc_bounds`

Applied to constrain foreign associated types.

```rust
trait Trait: Deref<Target: PartialEq + ?Sized> {}
```

Rust compiler, at the moment of writing this RFC, treats this form differently from the following:

```rust
trait Trait: Deref
where
    Self::Target: PartialEq + ?Sized
{
}
```

# Drawbacks
[drawbacks]: #drawbacks

- It may make reading source files of crates harder, as the reader should first look at the top of the crate to see the defaults, and then remember them. It may increase cognitive load.
- It may take some time for ecosystem around the language to fully adapt `!Trait` .

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design is simple yet powerful because it offers a backward-compatible way to evolve the language.

The impact of not accepting this RFC is that language features requiring types like `!Forget`, `MustMove`,
[`!AlignSized`] and many others will not be accepted.

[`!AlignSized`]: https://internals.rust-lang.org/t/pre-rfc-allow-array-stride-size/17933

## Alternative syntax
[alternative-syntax]: #alternative-syntax

We may have a single macro to declare all bounds:

```rust
declare_default_bounds! {
    generic: Sized, ?Forget, PartialEq;
    trait: ?Sized, ?Forget, PartialEq;
    assoc: Sized, ?Forget, PartialEq;
    foreign_assoc: ?Sized, ?Forget, PartialEq;
};
```

Or have separate macros for this:

```rust
declare_default_generic_bounds!(Sized, ?Forget, PartialEq);
declare_default_trait_bounds!(?Sized, ?Forget, PartialEq);
declare_default_assoc_bounds!(Sized, ?Forget, PartialEq);
declare_default_foreign_assoc_bounds!(?Sized, ?Forget, PartialEq);
```

## Use similar strategy of foreign associated types defaults, but over edition

It may be possible to use the same trick over an edition for traits that we want to remove from defaults. In the case of `Forget`, we may set default bound for crates of edition 2024 and earlier, and lift it for editions after 2024. In terms of this RFC, it would mean that editions would have different presets of default bounds, while users would not be able to manipulate them manually.

# Prior art
[prior-art]: #prior-art

- https://github.com/rust-lang/rust/pull/120706
- https://rust-lang.github.io/rfcs/0546-Self-not-sized-by-default.html

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- [ ] Maybe add shortcut for setting all 4 at once?
- [ ] Maybe instead special case `Sized`, while all other traits will be set with 1 macro instead of 4?
- [ ] Syntax
- [ ] How to display it in Rustdoc
- [ ] Should we allow default `!` bounds? What would it mean?
- [ ] 4 kinds seems too much... Maybe merge generics and local assocs? I was modelling after `Sized`, that's why there is 4 of them currently.
- [ ] Maybe use the term "implicit" instead of "default".

# Shiny future we are working towards

Less backward compatibility burden and more freedom to fix old mistakes, and propose new exciting features.
