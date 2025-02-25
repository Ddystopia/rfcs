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

A lot of new features (see [#use-cases](#use-cases)) require breaking old code by removing long-established assumptions like `size = stride` or the ability to skip the destructor of a type. To avoid breaking the code, they create a new trait representing an assumption and then define their feature as types that do not implement this trait. Here `?Trait` bounds come in - old code has old assumptions, but new code can add `?Trait` to opt out of them and support more types.

It is also important to note that in most cases those assumptions are not actually exercised by generic code, they are just already present in signatures - rarely code needs `size = stride`, or to skip the destructor (especially for a foreign type).

## The problem
[problem-of-default-bounds]: #problem-of-default-bounds

Quotes from "Size != Stride" [Pre-RFC thread](https://internals.rust-lang.org/t/pre-rfc-allow-array-stride-size/17933):

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
- `Size != Stride` is a [frequently requested feature][freaquently-requested-features-size-neq-stride], but it is [fundamentally backward-incompatible change that requires `?AlignSized` bound][size-neq-stride-backward-incompatibe].
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
#![default_generic_bounds(?Forget, PartialEq)]
```

The following example demonstrates how the compiler will understand the code. `PartialEq` is used just for illustration purposes. In reality, only a special set of traits would be allowed and would grow with new "breaking" traits, like `Forget`. `PartialEq` would not be one of them.

```rust
#![default_generic_bounds(?Forget, PartialEq)]

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

trait Trait: Deref<Target: ?Forget + PartialEq>
{
    type Assoc: PartialEq;
}

struct Qux;
struct Foo<T: ?Forget + PartialEq>(T);
struct Bar<T: ?Forget + ?PartialEq>(T);
struct Baz<'a, T>(T, &'a T::Target, T::Assoc)
where
    T: ?Forget + PartialEq,
    T: Trait<Target: ?Forget + PartialEq, Assoc: Forget + PartialEq>
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
```

2. Resolve any compilation errors by explicitly adding `+ Forget` where needed.

3. Optionally: Recurse into your dependencies, applying the same changes as needed.

Crates using `unsafe` code should beware of `ptr::write` and other unsafe ways of skipping destructors.

## Implications on the libraries

### Relax generic bound on public API

For migrated users it is equivalent to semver's `minor` change, while not migrated uses will observe it as `patch` change.

### Weakening associated type bound and `Self` bound in traits

Bounds for associated types and `Self` in traits would be weakened in respect to the new traits from the start:

```rust
trait Foo: ?Trait {
    type Assoc: ?Trait;
}
```

This change would not be observable for not migrated crates, because `default_generic_bounds` would default to `Trait`. But if users start migrate before libraries, they will not lock them into old bounds.

```rust
#![default_generic_bounds(?Forget)]

async fn foo<T: other_crate::Trait>(bar: T) {
    let fut = bar.baz();
    // Compiler will emit an error, as `fut` maybe `!Forget`, because we set `default_generic_bounds`
    // to `?Forget`, and `default_assoc_bounds` in `other_crate` is already `?Forget`. Otherwise it
    // would have been a breaking change for `other_crate` to make future provided by `baz` `!Forget`,
    // as this code would've compiled now but not in the future.
    core::mem::forget(fut);
}

// Libary that has not migrated yet.
mod other_crate {
    trait Trait {
        async fn baz();
    }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Introduce new trait level attibute: `default_generic_bounds` used to (non-exhaustively) enumerate overwrides of defaults for different types of bounds. Only a special set of traits would be allowed and would grow with new "breaking" traits, like `Forget`.

Every trait would initally have its unique default. In practice, bounds for all traits that are stable at the date of RFC except `Sized` would default to `?Trait`. For new "breaking" traits, default would be `Trait`, except bounds for `Self` in traits and associated types in traits.

[^trait-not-sized-by-default]: https://rust-lang.github.io/rfcs/0546-Self-not-sized-by-default.html

## Desugaring

### `default_generic_bounds`

Applied for generic parameters.

```rust
fn foo<T: PartialEq + Sized>() {}
struct Bar<T: PartialEq + Sized>(T);
trait Baz<T: PartialEq + Sized> {
    type Qux<U: PartialEq + Sized>;
}
trait Trait: Deref
where
    Self::Target: PartialEq + ?Sized
{
}

fn bar<T: Async>()
    where T::method(..): PartialEq + Sized
{}

trait Async {
    async fn method();
}
```

# Drawbacks
[drawbacks]: #drawbacks

- It may make reading source files of crates harder, as the reader should first look at the top of the crate to see the defaults, and then remember them. It may increase cognitive load.
- It may take some time for the ecosystem around the language to fully adapt `!Trait`, but it will not be a breaking change.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design is simple yet powerful because it offers a backward-compatible way to evolve the language.

The impact of not accepting this RFC is that language features requiring types like `!Forget`, `MustMove`,
[`!AlignSized`] and many others will not be accepted.

[`!AlignSized`]: https://internals.rust-lang.org/t/pre-rfc-allow-array-stride-size/17933

## Add fine-grained attributes
[split]: #split

We may have four attributes: `default_generic_bounds`, `default_foreign_assoc_bounds`, `default_trait_bounds` and `default_assoc_bounds` for more fine-grained control over defaults. For example, `Sized` has following defaults:

```rust
#![default_generic_bounds(Sized)]
#![default_trait_bounds(?Sized)]
#![default_assoc_bounds(Sized)]
#![default_foreign_assoc_bounds(?Sized)]
```

Previous version of this RFC was exactly this, you can read it [here](https://github.com/Ddystopia/rfcs/blob/leak-marker-trait-and-local-default-bounds/text/0000-local-default-generic-bounds.md).

## Alternative syntax
[alternative-syntax]: #alternative-syntax

We may have a single macro to declare all bounds:

```rust
declare_default_bounds! { Sized, ?Forget, PartialEq };
```

## Use similar strategy of foreign associated types defaults, but over edition

It may be possible to use the same trick over an edition for traits that we want to remove from defaults. In the case of `Forget`, we may set default bound for crates of edition 2024 and earlier, and lift it for editions after 2024. In terms of this RFC, it would mean that editions would have different presets of default bounds, while users would not be able to manipulate them manually.

# Prior art
[prior-art]: #prior-art

- https://github.com/rust-lang/rust/pull/120706
- https://rust-lang.github.io/rfcs/0546-Self-not-sized-by-default.html

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- [ ] Syntax
- [ ] How to display it in Rustdoc
- [ ] Should we allow default `!` bounds? What would it mean?
- [ ] Maybe use the term "implicit" instead of "default".
- [ ] Should we allow `Sized`.
- [ ] Maybe have 4 different attributes for more fine-grained control?

# Shiny future we are working towards

Less backward compatibility burden and more freedom to fix old mistakes, and propose new exciting features.
