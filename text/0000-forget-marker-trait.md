- Feature Name: `forget_marker_trait`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

<!-- todo: Replace with RFC PR later -->
[`local_default_bounds`]: https://github.com/Ddystopia/rfcs/blob/leak-marker-trait-and-local-default-bounds/text/0000-local-default-generic-bounds.md

# Summary
[summary]: #summary

Add a `Forget` marker trait indicating whether is it safe to skip the destructor before the type exits the scope and basic utilities to work with `!Forget` types. Introduce a seamless migration route for the standard library and crates.

# Motivation
[motivation]: #motivation

Many readers may find the biggest problem with `Forget` to be migration.
RFC's confidence is taken from the fact that migration can be done easily. See [#migration](#migration) section for details.

Back in 2015, the decision was made to remove the `Drop` guarantee, making every type implicitly implement `Forget`. All APIs in `std` could've been preserved without it. Only one of them needed to be changed. Today is 2025, and some things changed, and old reasoning is no longer true.  This RFC is not targeted at resource leaks in general but is instead focused on allowing a number of APIs to become safe.

## What are RAII guards? [^raii]
[raii-guards]: #raii-guards

[^raii]: https://rust-unofficial.github.io/patterns/patterns/behavioural/RAII.html

RAII is a useful pattern for ensuring resources are properly deallocated or finalized. We can make use of the borrow checker in Rust to statically prevent errors stemming from using resources after finalization takes place.

The core aim of the borrow checker is to ensure that references to data do not outlive that data. The RAII guard pattern works because the guard object contains a reference to the underlying resource and only exposes such references. Rust ensures that the guard cannot outlive the underlying resource and that references to the resource mediated by the guard cannot outlive the guard. To see how this works it is helpful to examine the signature of deref without lifetime elision:

```rust
fn deref<'a>(&'a self) -> &'a T {
    //..
}
```

The returned reference to the resource has the same lifetime as self ('a). The borrow checker therefore ensures that the lifetime of the reference to T is shorter than the lifetime of self.

## What is a proxy RAII guard?
[proxy-raii-guards]: #proxy-raii-guards

`thread::scoped` is special because it uses the RAII guard as a proxy to represent other values, but this proxy is not used to access those values. Instead, we are trusting that the borrow checker will ensure that the guard cannot outlive those values, and therefore that joining the thread in the guard's destructor is enough to ensure that the spawned thread is no longer running. [^proxy-raii-guard-source]

[^proxy-raii-guard-source]: https://github.com/rust-lang/rfcs/pull/1084#issuecomment-96875651

### Why is the proxy RAII guard gone?
[proxy-raii-guards-leakpokaplipse]: #proxy-raii-guards-leakpokaplipse

Back in 2015 [leakpocalypse] happened and a question was placed before the language: should we make skipping destructors safe or not? [PPYP] allows data structures to provide RAII guards, while being resilient to skipping the destructor. The only use case in std that cannot be expressed without destructor always running was `JoinGuard`, [which later got replaced too](thred-scope-doc).

[leakpocalypse]: https://github.com/rust-lang/rust/issues/24292
[PPYP]: https://cglab.ca/~abeinges/blah/everyone-poops/
[thred-scope-doc]: https://doc.rust-lang.org/std/thread/fn.scope.html

Instead of having a guarantee of the destructor running we can take a closure/callback instead of returning a guard object:

```rust
fn something_with_clean_up(f: impl FnOnce(Foo)) {
    // Setup
    f(Foo);
    // Clean up. It is *guaranteed* to run, like destructors for variables in `Setup`.
}
```

Thus, there was no point in redesigning the language and delaying Rust 1.0, practically all APIs and patterns could've been safely expressed without destructors always running, so making `std::mem::forget` safe was a good decision at the time.

### What is different
[what-is-different]: #what-is-different

Edition 2018 introduced `sync` Rust. But as turned out, nuances in its design conflicted with an earlier decision. All `async` calls are essentially constructors for state machines, which borrow some resources from outside or directly own them. It is on the user to poll those state machines to completion. `!Forget` patterns could've been expressed by other means with sync Rust (like taking a callback instead of returning a guard or PPYP), but with `async`, anything turns directly into `impl Future + use<'a>`, which is equivalent to the RAII guard.

Various OS or C/C++ APIs cannot be made `async` without performance or ergonomics costs. PPYP can work for `Drain<'a>`, but not for `io_uring`. As long as the future directly owns (or is `'static`) all data it is accessing `Pin` guarantees are sufficient. Otherwise, there is no way to make a sound API.

Let's try to translate the previous example, a widely used pattern, to sync Rust.

```rust
fn something_with_clean_up(f: impl FnOnce(Foo)) {
    // setup
    f(Foo);
    // clean up
}

fn main() {
    something_with_clean_up(|foo| {
        foo.bar();
    });
    // rest of the code...
}
```

As you can see, after calling `something_with_clean_up`, the control flow is passed to the library. The rest of the user's code *cannot* continue executing before `something_with_clean_up` performs a cleanup (assuming unwinding is handled properly).


```rust
async fn something_with_clean_up(f: impl AsyncFnOnce(Foo)) {
    // setup
    f(Foo).await;
    // clean up
}

async fn main() {
    something_with_clean_up(|foo| {
        foo.bar();
    }).await;
    // rest of the code...
}
```

In this code snipped we added `async` modifiers to our functions, as well as `await`. You may think that cleanup will be done, but in reality, it is not guaranteed. All `async` calls are turned into structs - like RAII guards we talked about earlier:

```rust
async fn something_with_clean_up(f: impl AsyncFnOnce(Foo)) {
    // setup
    f(Foo).await;
    // clean up
}

async fn main() {
    let fut = something_with_clean_up(|foo| {
        foo.bar();
    });
    {
        let pinned = Box::pin(fut);
        poll_fn(|cx| Poll::Ready(_ = pinned.poll(cx))).await;
        forget(pinned);
    }
    // rest of the code...
}
```

The library is only taking control flow in between `await` points. Here, future is pinned and [Pin]'s [drop guarantee] is met (boxed future remains allocated for `'static`), but clean up cannot run. Thus, APIs that require any cleanup for safety can be expressed in `sync` Rust, but not in `async` Rust, making `async` less attractive, as the operating system and other C/C++ libraries *cannot* be used efficiently, ergonomically, and safely.

[drop guarantee]: https://doc.rust-lang.org/std/pin/#drop-guarantee

Another important observation that we can make is that `Pin`'s drop guarantee only applies to the memory of the `Future` itself. But if `Future` borrows a buffer, it *can* be deallocated or re-used before the `drop` of the `Future` is called. See [#connection-to-pin](#connection-to-pin).

[#reference-level-explanation](#reference-level-explanation).
[#connection-to-pin](#connection-to-pin).

## Examples of unsafe async APIs that can be allowed in sync Rust
[example-safe-sync-unsafe-async]: #example-safe-sync-unsafe-async

### Async spawn
[example-async-spawn]: #example-async-spawn

Example from the ecosystem: [spawn_unchecked](spawn_unchecked-example-doc)

[spawn_unchecked-example-doc]: https://docs.rs/async-task/latest/async_task/fn.spawn_unchecked.html.

With the `Forget` trait we can make that API safe:

```rust
struct TaskHandler<'a>(u64, PhantomNonForget, PhantomData<&'a ()>);
// Or `struct TaskHandler<'a>(u64, PhantomNonForget<&'a ()>)`;

impl Drop for TaskHandler<'_> {
    fn drop(&mut self) {
        if let Some(mut mutex) = GLOBAL.get(self.0) {
            // We can block in async context as this mutex is held during the `poll`, which should return in a timely manner.
            let fut = mutex.lock();
            // cancel the future and call its drop handler
            drop(fut.take())
        }
    }
}

fn spawn<'a>(fut: impl IntoFuture + 'a) -> TaskHandler<'a> {
    GLOBAL.spawn(fut)
}
```

### Async DMA
[example-async-dma]: #example-async-dma

DMA stands for Direct Memory Access and it’s a peripheral used for transferring data between two memory locations in parallel to the operation of the core processor. For the purposes of this example, it can be thought of as `memcpy` in parallel to any other code.

Let's say that `Serial::read_exact` triggers a DMA transfer and returns a future that will resolve on completion. It would be safe if we were to block on this future (basically passing control flow to the future itself), but we may instead trigger undefined behavior with `forget`:

```rust
fn start(serial: &mut Serial) {
    let mut buf = [0; 16];

    // not `unsafe`!
    mem::forget(serial.read_exact(&mut buf));
}

fn corrupted() {
    let mut x = 0;
    let y = 0;

    // do stuff with `x` and `y`
}

start(&mut serial);
// `DMA` keeps writing to `buf`, which is on the stack. `x` and `y` live on the stack too,
// so they will be corrupted.
corrupted();
```

See [blog.japaric.io/safe-dma] for more.

[blog.japaric.io/safe-dma]: https://blog.japaric.io/safe-dma/

### GPU
[example-async-cuda]: #example-async-cuda

[`async-cuda`], an ergonomic library for interacting with the GPU asynchronously. GPU is just another I/O device (from the point of view of the program), the async model fits surprisingly well. But, this library enforces `!Forget` via documentation requirements.

[`async-cuda`]: https://crates.io/crates/async-cuda

> Internally, the Future type in this crate schedules a CUDA call on a separate runtime thread. To make the API as ergonomic as possible, the lifetime bounds of the closure (that is sent to the runtime) are tied to the future object. To enforce this bound, the future will block and wait if it is dropped. This mechanism relies on the future being driven to completion, and not forgotten. This is not necessarily guaranteed. Unsafety may arise if either the runtime gives up on or forgets the future, or the caller manually polls the future, then forgets it.

### `take_mut`

The async version of [`take_mut`] cannot be created as it relies on cleanup code to abort the program.

[`take_mut`]: https://docs.rs/take_mut/latest/take_mut/

### `io_uring`
[example-async-io_uring]: #example-async-io_uring

`io_uring` is another API that needs `!Forget` in order to function properly. There are attempts at making safe wrappers like [`ringbahn`], which introduces an internal buffer, or [`tokio_uring`], that requires passing an ownership of the target buffer.

[`rio`] took an approach like `async-cuda`, implicitly making its futures `!Forget` via documentation.

> `rio` aims to leverage Rust's compile-time checks to be misuse-resistant compared to io_uring interfaces in other languages, but users should beware that use-after-free bugs are still possible without `unsafe` when using `rio`. `Completion` borrows the buffers involved in a request and its destructor blocks to delay the freeing of those buffers until the corresponding request has been completed, but it is considered safe in Rust for an object's lifetime and borrows to end without its destructor running, and this can happen in various ways, including through `std::mem::forget`. Be careful not to let completions leak in this way, and if Rust's soundness guarantees are important to you, you may want to avoid this crate.

[`ringbahn]`: https://github.com/ringbahn/ringbahn/
[`tokio_uring`]: https://docs.rs/tokio-uring/latest/tokio_uring/
[`rio`]: https://lib.rs/crates/rio

### C/C++ bindings + async do not work well together
[example-async-c-cpp-bindings]: #example-async-c-cpp-bindings

It is very common for C/C++ APIs to require some cleanup. It is not an issue for `sync` rust, as wrappers can just take a closure/callback and ensure that cleanup. But all `async` calls are transformed into `impl Future + use<'a>`, not passing control flow to the wrapper. `io_uring` and `async-cuda` fall into that category too. For embedded/kernel development this issue is even worse, as you often cannot afford an allocation due to lack of resources or complex locking, making borrows your only option and making `Pin`'s drop guarantee not useful for you.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The core goal of `Forget` trait, as proposed in that RFC, is to bring back the "Proxy Guard" idiom for non-static types, with `async` being the primary motivation.

## What does `!Forget` mean?
[what-not-forget-mean]: #what-not-forget-mean

If any resources are borrowed by some type `T: !Forget`, they will remain borrowed until `T` is dropped. See a more precise description in [#reference-level-explanation](#reference-level-explanation).

```rust
let mut resource = [0u8; 1024];

let borrower: Borrower<'_> = Borrower::new(&mut resource);

std::mem::forget(borrower); // Violation of the unsafe contract - `resource` is no longer borrowed, so repurposing protected memory is safe.

let first_byte = resource[0]; // Potential UB
```

## How is `Forget` related to `Pin`?
[connection-to-pin]: #connection-to-pin

Both `Forget` and `Pin` concepts serve a similar purpose - guaranteeing that some memory is not moved or repurposed. How `Forget` does it? If any resource is borrowed, you cannot take `&mut` reference to it, as it would be aliased by `!Forget` type that is borrowing from it. Before `!Forget` type goes out of scope, removing the borrow, its drop handler must be executed, just like `Pin`'s [drop guarantee]. So `!Unpin` protects directly owned memory, while `!Forget` protects *borrowed* memory.

With `Forget`, some authors may have the option to borrow the data instead of owning it, making their futures `Unpin`, but `!Forget`.

## Core problem

Consider that example

```rust
fn spawn<F: IntoFuture>(fut: F) -> JoinHandle<F> {
    // store the future in global storage
}

fn main() {
    let mut buf = [0u8; 64];
    let fut = async {
        let mut i = 0;
        loop {
            // `&mut` to `buf`.
            buf[i] = i;
            i = (i + 1) % 64;
            yield_now().await;
        }
    };

    let handle = spawn(fut);
    std::mem::forget(handle);
    // `fut` might still be running in the background, but `buf` is no longer protected by the borrow checker.

    // Undefined Behavior - aliasing mutable reference
    let fourth = buf[4];
}
```

In this case, `handle` borrows from `buf`, but the code that operating of `buf` is not directly tied to `handle`, but operates independently of it. Because of this, even if we pin `handle`, we still can *remove the borrow* (by ending the lifetime of `handle`) on `buf`, even if `JoinHandle`'s memory remains available (`forget(Box::pin(handle))`).

Functions having signatures with weakening can skip the drop handler of a type. The following function is an example of a weakening function - after it is called, the borrow checker assumes that the lifetime of `T` has ended, as well as all borrows held by `T`.

```rust
fn weakener<T>(foo: T) -> i32 {
    std::mem::forget(T);
    0
}
```


Currently, many APIs are forced into using `'static` bounds, which is one of the pain points users are reporting about `async` Rust, together with `Send` issues.

## Not only forgets
[channels-unsoundness]: #channels-unsoundness

There exists a way to exploit the old `thread::spawn` API without any memory leaks! We can move `JoinHandle` inside the thread it is meant to protect, creating a kind of cycle:

```rust
use std::{
    hint::black_box,
    marker::PhantomData,
    sync::{Arc, Mutex},
};

struct JoinHandle<'a>(PhantomData<&'a ()>);

impl Drop for JoinHandle<'_> {
    fn drop(&mut self) {}
}

fn spawn<'a, F>(_f: F) -> JoinHandle<'a>
where
    F: FnOnce() -> (),
    F: Send + 'a,
{
    todo!()
}

fn main() {
    let arc1 = Arc::new(Mutex::new(None));
    let arc2 = arc1.clone();

    let mut buf = [0; 1024];
    let buf_ref = &mut buf;

    let handle = spawn(move || {
        let _handle = arc2.lock().unwrap().take();
        for _ in 0..100000 {
            black_box(&mut *buf_ref);
        }
        drop(arc2);
    });

    arc1.lock().unwrap().replace(handle);
    drop(arc1);

    // aliased `&mut`
    buf[0] = 1;
}
```

In this code, no memory is leaked, and `JoinHandle`'s destructor is not skipped. Many kinds of channels, including rendezvous channels, have signatures replicable with reference counters - they are susceptible to this exploit as well:

```rust
fn main() {
    let (tx, rx) = std::sync::mpsc::channel();

    let mut buf = [0; 1024];
    let buf_ref = &mut buf;

    let handle = spawn(move || {
        let _handle = rx.recv().unwrap();
        for _ in 0..100000 {
            black_box(&mut *buf_ref);
        }
        drop(_handle);
    });

    tx.send(handle);
    drop(tx);

    buf[0] = 1;
}
```

### Solution for message passing of `!Forget` types.
[solution-to-self-referential-problem]: #solution-to-self-referential-problem

One might speculate and try to fix some holes, for example by making `JoinHandle: !Send`, but this can only count as a workaround. By looking at the depth of an issue we can see, that `Forget` is generally incompatible with `Rc`, as well as other APIs that can be expressed with its signature. In the example earlier, the borrow checker cannot see a connection between `rx` and `tx` - when `tx` is dropped, `buf` is no longer borrowed. What if retained such a connection?

```rust
fn main() {
    let mutex = Mutex::new(None);
    let mutex_ref = &mutex;

    let mut buf = [0; 1024];
    let buf_ref = &mut buf;

    let handle = spawn(move || {
        let _handle = mutex_ref.lock().unwrap().take();
        for _ in 0..100000 {
            black_box(&mut *buf_ref);
        }
    });

    mutex.lock().unwrap().replace(handle);

    buf[0] = 1;
}
```

And we got a compiler error, preventing the unsoundness:

```rust
 1  error[E0597]: `mutex` does not live long enough
   --> src/main.rs:23:21
    |
 22 |     let mutex = Mutex::new(None::<JoinHandle<'_>>);
    |         ----- binding `mutex` declared here
 23 |     let mutex_ref = &mutex;
    |                     ^^^^^^ borrowed value does not live long enough
 ...
 38 | }
    | -
    | |
    | `mutex` dropped here while still borrowed
    | borrow might be used here, when `mutex` is dropped and runs the destructor for type `Mutex<Option<JoinHandle<'_>>>`

 2  error[E0506]: cannot assign to `buf[_]` because it is borrowed
   --> src/main.rs:37:5
    |
 26 |     let buf_ref = &mut buf;
    |                   -------- `buf[_]` is borrowed here
 ...
 37 |     buf[0] = 1;
    |     ^^^^^^^^^^ `buf[_]` is assigned to here but it was already borrowed
 38 | }
    | - borrow might be used here, when `mutex` is dropped and runs the destructor for type `Mutex<Option<JoinHandle<'_>>>`
```

This example is exactly like the first example with `Arc`, but uses references instead - we are allowed to pass them with `JoinHandle: !Forget`. But what with channels? There are not so many examples in the ecosystem that follow this approach in the API, as it is not `'static`, but there are some:

```rust
fn main() {
    let mut queue = heapless::spsc::Queue::<_, 2>::new();
    let (mut tx, mut rx) = queue.split();

    let mut buf = [0; 1024];
    let buf_ref = &mut buf;

    let handle = spawn(move || {
        let _handle = rx.dequeue();
        for _ in 0..100000 {
            black_box(&mut *buf_ref);
        }
    });

    tx.enqueue(handle);
    drop(tx);

    buf[0] = 1;
}
```

This code fails to compile too. Why? Because `tx` is connected to `queue` and `rx` is connected to `queue` too. After `tx` is dropped, `buf` remains borrowed by `handle` until the lifetime of `queue`. Can we drop `queue` then? We can't, because `rx` still borrows it, through the `handle`. This is clearly a cycle, and the borrow checker is able to catch it this time.

This means that to use message-passing with `!Forget` types, API authors must rely on lifetimes more - because `Forget` types are all about lifetimes. Looking at the example above, `rx` cannot be passed to the traditional `spawn`, because of the `F: 'static` requirement. But `thread::scope` is fine with it - as well as async `scope` is, with the future itself being `!Forget`. Note that rendezvous channels can be soundly expressed using that API and `PhantomData`.

## Traditional combinators and patterns
[traditional-workflows]: #traditional-workflows

Async combinators with `join`, `race`, or `merge` semantics will continue to work as they do. If some future passed into them is `!Forget`, their future becomes `!Forget` too. `Arc` cannot be used with `!Forget` types, but the need for `Arc`, [which is quite a pain point](ergonomic-refcounting), will decrease, as users will be able to spawn with references directly.

[ergonomic-refcounting]: https://github.com/rust-lang/rfcs/pull/3680

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This new auto trait is added to the `core::marker` and `std::marker` modules:

```rust
pub unsafe auto trait Forget { }
```

Unsafe code authors can rely on the fact that memory borrowed by `!Forget` types are not reused or invalidated until the drop (just like `Pin`'s [drop guarantee], but with indirection).  Note that for `T: 'static` we don't have to run the destructor to fulfill this guarantee, as `'static` borrows can be assumed to be valid indefinitely (like with [`Pin::static_ref`]).

[`Pin::static_ref`]: https://doc.rust-lang.org/std/pin/struct.Pin.html#method.static_ref

In practice, we disallow skipping the destructor of `!Forget` types before they exit the scope. Violation is not an immediate undefined behavior, but other code can rely on the destructor running, which can lead to undefined behavior down the road.

Type becomes `!Forget` if it directly contains `!Forget` member.

We should either allow `!Forget` types in statics or make all `'static` types `Forget` because it fulfills the unsafe guarantee and we can't enforce any code running before the program's abortion.

```rust
let mut resource = [0u8; 64];
let _unforget = Subsystem::execute(&mut resource);
std::process::abort(); // `resource` is (forcefully) borrowed for `'static`
resource[0] = 42; // unreachable
```

## Standard Library
[std]: #std

All APIs in the standard library should be migrated at once. With available migration strategies, there is no benefit in gradual migration, while it will greatly reduce the productivity of rustc developers by adding boilerplate and noise into the codebase. An audit must be performed to ensure which APIs must remain `Forget`. See [#migration](#migration) for more details.

No types in std will be changed to `!Forget`.

## `Copy`
[copy]: #copy

All types that implement `Copy` must implement `Forget` too.

## Unions
[unions]: #unions

Unions are always `Forget`. All members of `union` must be `Forget`, but it is already covered by other rules and does not need to be enforced.

## API changes
[library-api-changes]: #library-api-changes

- `Rc`/`Arc` - all APIs for construction,  except the new `Rc::new_unchecked` method, only exist for `T: Forget` types. In the future we *may* allow safe constructors for `T: ?Forget + 'static` (resources are borrowed for `'static`, it fulfills the guarantee we are giving to the unsafe code) and something along the lines of `T: ?Forget + Freeze` (to forbid cycles), RFC author is not familiar enough with interior mutability questions.
- `ManuallyDrop<T>` always implements `Forget`, regardless of the `T`. `ManuallyDrop::new` is available for types with `T: Forget`.  New unsafe method `ManuallyDrop::new_unchecked`, available for `T: ?Forget`, is introduced. We may add a safe constructor with `T: ?Forget + 'static`, as we allow forgetting in statics.
- `Box::<T>::into_ptr` is available only for `T: Forget`. As for `T: !Forget` users should `ManuallyDrop::new_unchecked` and take the pointer via `&raw mut`. It will still be allowed to pass this pointer to `Box::from_ptr`.
- `Box::<T>::forget` is available only for `T: Forget`.
- `forget_unchecked`, a new unsafe function, is added to forget `T: ?Forget` types. It is a wrapper around `ManuallyDrop::new_unchecked`, just as `forget` is a wrapper around `ManuallyDrop::new`.
- `PhantomNonForget` is a `!Forget` ZST for types to become `!Forget`.
- `Vec::drain` is available only for `T: Forget` types. A new method might be added to work with `T: ?Forget`.
- APIs like `std::sync::mpsc::Sender::send` are available only for `T: Forget`.
- Possibly new channels should be introduced, that are compatible with `T: !Forget` types too.
- [ ] `Vec::set_len` is available only for `T: Forget` types, to not create a footgun for `unsafe` code in the wild. Maybe a new method should be added to support `T: ?Forget`.

## Migration
[migration]: #drawbacks

### Migration using [`local_default_bounds`] RFC
[local-defaults-migration]: #local-defaults-migration

`local_default_bounds` RFC can help with making a migration smoother and does not require an edition. In a nutshell, it allows users to override default trait bounds, for example removing the `Sized` default, or adding the `MyFavouriteTrait` default. 

In terms of `local_default_bounds` RFC, together with adding the `Forget` trait, `default_trait_bounds` `default_assoc_bounds` should become `?Forget` instead of `Forget`. This is not observable for any code that is not opting into using `Forget` explicitly, as `default_generic_bounds` and `default_foreign_assoc_bounds` are still `Forget`. It will be discussed later in [#semver-and-ecosystem](#semver-and-ecosystem).

As discussed in [#semver-and-ecosystem](#semver-and-ecosystem), libraries adopting `?Forget` signatures will be a minor semver change at most. Thus, migration to `?Forget` would be equivalent to the currently accepted and stable `const fn`. Libraries are already adopting `const fn` and there is no notion against using `const` functions or ecosystem split - the ecosystem is migrating and PRs are being merged, making more and more functions `const`.

#### Not interested in migration crates
[no-local-defaults-migration]: #no-local-defaults-migration

Some crates may refuse to migrate due to being unmaintained, the only difference is that for downstream crates their signatures would be filled with `T: Forget`. This is only natural, as those crates were written with that assumption as if they manually put `T: Forget` on their signatures. Some automatic methods to determine that function can accept `Forget` types is not feasible because it can only work if only safe code is interacting with `T` and will be a semver hazard.

If the crate is maintained, however, migration should not be difficult.

#### `#![forbid(unsafe)]` crates
[safe-local-defaults-migration]: #safe-local-defaults-migration

1. Set the appropriate bounds:

```rust
#![default_generic_bounds(?Forget)] // can be with `cfg_attr`
#![default_foreign_assoc_bounds(?Forget)]
```

2. Resolve any compilation errors by explicitly adding `+ Forget` where needed.

3. Optionally: Recurse into your dependencies, applying the same changes as needed. Most probably you will use `!Forget` types with well-maintained crates providing combinators or containers.

#### For crates with `unsafe` code (like `libcore`)
[unsafe-local-defaults-migration]: #unsafe-local-defaults-migration

1. Set the appropriate bounds:

```rust
#![default_generic_bounds(?Forget)]
#![default_foreign_assoc_bounds(?Forget)]
```

2. Audite your codebase to work properly with `!Forget` types.

3. Resolve any compilation errors by explicitly adding `+ Forget` where needed.

4. Optionally: Recurse into your dependencies, applying the same changes as needed.

#### Semver and compatibility, ecosystem splitting
[semver-and-ecosystem]: #semver-and-ecosystem

This approach is targeted at minimizing problems between different crates in the ecosystem. For any library, opting into using `Forget` and accepting those types will be a minor semver change.

Earlier it was stated that `default_trait_bounds` and `default_assoc_bounds` should become `?Forget` instead of `Forget`. This is due to an important case. If a user of the library updated earlier than the library, then without that change it will observe that associated types of traits are `Forget`, so it would be a breaking change for the library to lift that constraint in the future. But now, the user will observe `?Forget`, thus it cannot rely on them being `Forget`. But for users that did not migrate, as well as the library itself, it will not be observable due to `default_generic_bounds` and `default_foreign_assoc_bounds` still being `Forget`.

```rust
#![default_generic_bounds(?Forget)]
#![default_foreign_assoc_bounds(?Forget)]

async fn foo<T: other_crate::Trait>(bar: T) {
    let fut = bar.baz();
    // Compiler will emit an error, as `fut` maybe `!Forget`, because we set `default_foreign_assoc_bounds`
    // to `?Forget`, and `default_assoc_bounds` in `other_crate` is already `?Forget`. Otherwise it
    // would have been a breaking change for `other_crate` to make future provided by `baz` `!Forget`,
    // as this code would've compiled now but not in the future.
    core::mem::forget(fut); 
}

// `other_crate`
mod other_crate {
    trait Trait {
        async fn baz();
    }
}
```

### Migration over the edition, with a mask
[edition-migration-with-mask]: #edition-migration-with-mask

We can have a satisfactory migration experience even without any additional language features. We may have editions <= 2024 have `Forget` as default, and editions after 2024 have `?Forget` as default.

While will not split the ecosystem, will require everyone to make migration a migration just as in the [`local_default_bounds`] solution. It can be automated for `#![forbid(unsafe)]` crates.

### Manually mark almost every generic bound as `?Forget`
[manual-migration]: #manual-migration

This is a manual, tedious process, it will pollute the codebases with boilerplate and make them look awful and, therefore awful to maintain. Even more, associated types must remain `Forget`, as some code might rely on it, and removing that guarantee would be a breaking change. So this doesn't seem like a practical solution, we need a mechanism for some crates to observe `Forget` bound, and for others to not.

# Drawbacks
[drawbacks]: #drawbacks

## Migration
[drawbacks-migration]: #drawbacks-migration

If [`local_default_bounds`] is accepted, migration would be practically seamless, as described in [#migration](#migration). Even if it's not accepted, less seamless but still acceptable solution would be a change over edition.

## Message Passing
[drawbacks-message-passing]: #drawbacks-message-passing

A traditional approach to message-passing cannot be applied to `!Forget` types - slightly different APIs should be developed, preserving a
lifetime connection between `tx` and `rx` handles.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

All types were assumed to be `!Forget` in Rust's early days, and then it was changed in hurry. They flow naturally out of Rust's type system, do not clash with any preexisting concepts that do not directly involve forgetting and are very pleasant and intuitive to use, modulo migration. With the `Future` trait it became apparent that language directly lacks this feature, it is very simple and non-disturbing, so it's hard to find something that would fit that purpose better.

We can do nothing, but use cases just keep piling up.

The author of https://zetanumbers.github.io/book/myosotis.html is working on another approach to that problem, but it is not public yet.

# Prior Art
[prior-art]: #prior-art

- https://github.com/rust-lang/rfcs/pull/1084#issuecomment-96875651
- https://github.com/rust-lang/rfcs/pull/1094
- https://internals.rust-lang.org/t/forgetting-futures-with-borrowed-data/10824
- https://github.com/aturon/rfcs/blob/scoped-take-2/text/0000-scoped-take-2.md
- https://without.boats/blog/the-scoped-task-trilemma
- https://without.boats/blog/asynchronous-clean-up/
- https://zetanumbers.github.io/book/myosotis.html: an independent exploration of the same problem space with similar, but subtly different, conclusions.
- https://hackmd.io/@wg-async/S1Q6Leam0: a design meeting regarding the previous post

## Leakpocalypse
[leakpocalypse-prior-art]: #leakpocalypse-prior-art

- https://github.com/rust-lang/rfcs/pull/3680
- https://github.com/rust-lang/rust/issues/24292
- https://cglab.ca/~abeinges/blah/everyone-poops/
- https://github.com/rust-lang/rfcs/pull/1085
- https://doc.rust-lang.org/std/thread/fn.scope.html
- https://smallcultfollowing.com/babysteps/blog/2015/04/29/on-reference-counting-and-leaks/

## Usage of the pattern
[usage-prior-art]: #usage-prior-art

- https://doc.rust-lang.org/std/thread/struct.Builder.html#method.spawn_unchecked
- https://docs.rs/async-task/latest/async_task/fn.spawn_unchecked.html
- https://blog.japaric.io/safe-dma/
- https://docs.rs/async_nursery/latest/async_nursery/
- https://without.boats/blog/the-scoped-task-trilemma/
- https://docs.rs/async-scoped/latest/async_scoped/struct.Scope.html#method.scope_and_collect

## Miscellaneous
[misc-prior-art]: #misc-prior-art

- https://github.com/rust-lang/rfcs/issues/1111 
- https://tmandry.gitlab.io/blog/posts/2023-03-01-scoped-tasks/

## MustMove types
[must-move-prior-art]: #must-move-prior-art

- https://faultlore.com/blah/linear-rust/
- https://blog.yoshuawuyts.com/linear-types-one-pager/
- https://smallcultfollowing.com/babysteps/blog/2023/03/16/must-move-types/

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- [ ] Maybe the name `Forget` is misleading, as its core is around the `unsafe` guarantee of borrowed resources and the destructor. `IndirectPin`?
- [ ] Maybe force `impl Forget for T where T: 'static {}` and add a generic to the `PhantomNonForget`? Use cases and unsafe guarantee are fine with it, and we already allow `!Forget` in `static`.
- [ ] Maybe add `StaticForget<T: ?Forget + 'static>: Forget`.
- [ ] How does it interact with `&own`?
- [ ] Maybe make `Vec::set_len` available for `T: ?Forget`, but with a new unsafe precondition. Crates with unsafe code that are manually migrating to support `!Forget` would need to be aware of that change and verify/modify their unsafe code to work correctly with `!Forget` types, or manually restrain them to `Forget`.
- [ ] Which approach to migration should be followed?
- [ ] How should it interact with `async Drop`?

# Future possibilities
[future-possibilities]: #future-possibilities

This RFC will allow `async` Rust to come closer to sync ergonomics, but some code will not be able to reach this end goal and insert "abort bombs" into mandatory destructors. This is strictly better than today's status quo: `unsafe` in application code, you can work with it, but this is not ideal. A more robust approach would be the `Linear`/`MustMove`/`!Drop` types. This RFC makes a step towards more liveness guarantees, making them closer. As for the biggest problem - unwinding - with `async`, we have more choice over our behavior during unwinds. Even if we do not succeed with effects forbidding unwinding, the future containing linear type may catch any unwind during the poll and return `Poll::Pending`, potentially recovering - `async Drop` looks promising too.


