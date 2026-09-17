+++
title = "It is time to understand Pin!!"
date = 2026-09-04
slug = "understand-rust-pinning"
[taxonomies]
tags = ["programming","rust"]
+++

Pinning is notoriously one of those concepts that baffles many developers (myself included). Naturally, I thought it would be worth breaking down in a dedicated post. It took me a while to grasp the core intuition—largely because existing explanations feel fragmented. The goal of this post is to demystify Pinning, build up the mental model step-by-step, and make it click. So grab your favorite coffee, settle in, and let's dive in.<!--more-->



## The concept: Pinning in it simplest form !!!

When we talk about pinning an object, we simply mean anchoring it to a specific spot in memory and preventing it from being relocated. In other words, we want to stop an object from moving across different memory addresses over time (we will explore why shortly). You are free to inspect an object and read its state, but mutating or relocating it is strictly prohibited- much like an authoritarian developer ruling over a dystopian codebase.

How do we enforce this? By leveraging the compiler's strict rulebook. First, let's revisit the fundamental laws of borrowing when taking a reference:

1- You cannot move or modify an object while an active reference—shared or mutable—is held against it.
2- At any given moment, you can hold either a single mutable reference OR any number of immutable references.

Let's construct a minimal example to illustrate this mechanics:


```rust
fn main() {
    let mut a = 5;
    let mut b = 7;
    let locked_x = &x;      // now x is locked
    let y = x;              // ❌ you can't move object while you hold reference
    let z = &mut x;         // ❌ you can't have mutable reference
    mem::swap(&mut x, &mut b)
    drop(locked_x);
}
```

By holding a reference, we effectively prevent ownership of the underlying object from being transferred, while simultaneously blocking mutable references that could be used to swap or move it. Mission accomplished! While this mechanism feels almost like a trick, it works seamlessly- no runtime magic or complex compiler incantations required.

To put it concisely: Pinning is simply a mechanism that relies on ownership and borrow-checker rules to restrict access to a mutable reference. Full stop.

If this concept makes sense, congratulations!! you have unlocked the core principle behind pinning.


## What is moving objects ?

Under the hood, structs and objects are structured byte sequences residing in addressable memory cells. Moving an object simply entails copying its raw bytes from one memory location to another, while simultaneously transferring ownership and destructor responsibilities. Consider this instantiation:

```rust
let my_car = Car { /* ... */ }
```

Here, `my_car` resides at address `0x1000000000` with the following memory layout:

```rust
struct Car { // @ 0x1000000000
    width: f32, // @ 0x1000000000
    height: f32, // @ 0x1000000004
    length: f32, // @ 0x1000000008
    weight: f32, // @ 0x100000000C
}
```

Relocating this object shifts its base address to a new memory slot, such as `0x2000000000`:


```rust
struct Car { // @ 0x2000000000
    width: f32, // @ 0x2000000000
    height: f32, // @ 0x2000000004
    length: f32, // @ 0x2000000008
    weight: f32, // @ 0x200000000C
}

```



## Ok, what about self-referential struct

In safe Rust, struct fields cannot hold direct references to sibling fields within the same instance. The immediate barrier is construction: how do you initialize an internal reference to a field when the enclosing object hasn't even finished initializing?

However, an even deeper issue emerges during moves. Consider an object a placed at `0x1000000000`, containing a pointer `abc_ref` pointing directly to its internal field `abc`:


```rust
struct SelfReferring { // @ 0x1000000000
    abc: i32,        // @ 0x1000000000    => 67
    abc_ref: *const i32 // @ 0x1000000004 => points to abc @ 0x1000000000 (67)
}
```
Here, abc = 67, abc_ref = 0x1000000000, and *abc_ref = 67.
Now, suppose we transfer ownership of object `a` to new variable `b`:

```rust
let b = a
```
The underlying memory representation of `b` is moved to `0x2000000000`:

```rust
struct SelfReferring {  // @ 0x2000000000
    abc: i32,           // @ 0x2000000000    => 67
    abc_ref: *const i32 // @ 0x2000000004 => points to abc @ 0x1000000000 (????)
}

```
Notice the catch: while `abc` shifted to `0x2000000000`, `abc_ref` still points back to the stale address `0x1000000000`! Dereferencing `abc_ref` now leads to dangling pointers and undefined behavior. We could theoretically update pointer offsets on every move, but that would impose a significant runtime overhead. Or... is there a better way?

## Return to pinning

The alternative- and far more elegant- approach is to guarantee that moving the object is entirely impossible. We can achieve this by formalizing our earlier borrow-checker mechanism within Rust's type system:


```rust
pub struct MyPin<'a, T> {
    ptr: &'a T,
}

impl<'a, T> MyPin<'a, T> {
    pub fn new(rf: &'a T) -> Self {
        MyPin { ptr: rf }
    }
}

fn main() {
    let mut x = 5;
    let locked_x = MyPin::new(&x);
    // now those operations are invalid (compile error)
    //let x = x;
    //mem::swap(&mut x, &mut other);
    drop(locked_x);
}
```
Congratulations! You now understand the fundamental blueprint of Pin. Everything else on top of this is merely extra architectural polish.

## More enhancements
Right now, our `MyPin` abstraction is essentially a black hole: it locks the object away so thoroughly that no one can read its contents. To fix this, we can implement the Deref trait to restore safe read access:

```rust
impl<'a, T> Deref for MyPin<'a, T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        self.ptr
    }
}
```
Much better! We can now conveniently read `x` through `locked_x` without risking a memory move.

## What about other reference types?
So far, our `MyPin` holds a simple shared reference (&T). But what if we want to pin heap-allocated or smart pointer types like `Box`, `Rc`, or `Cow`..etc?

To achieve this, we can abstract MyPin over any pointer type that implements `Deref`:

```rust
pub struct MyPin<T> {
    ptr: T,
}


impl<T> MyPin<T>
where
    T: Deref,
{
    pub fn new(rf: T) -> Self {
        MyPin { ptr: rf }
    }
}

impl<T> Deref for MyPin<T>
where

    T: Deref,

{
    type Target = T::Target;
    fn deref(&self) -> &Self::Target {
        &self.ptr
    }
}
```

At this point, our custom implementation closely mirrors `std::pin::Pin`.

It is worth noting an important distinction here regarding Box: when you wrap a `Box<T>` in Pin, the Box container itself moves into the `Pin`, but the heap payload it points to remains immutably fixed at its memory address. Even if the Box on the stack is relocated, the underlying heap data stays pinned.

## Pin-able or Not pin-able, this is the question ?
The Rust language designers realized that this exact mechanism was indispensable for managing self-referential structures in Async Rust.

Under the hood, when the compiler encounters an async block or function, it transforms it into an anonymous state machine. Local stack variables that must persist across .await points are saved inside this state machine, naturally forming self-referential pointers across state boundaries.

However, enforcing strict unmovable semantics across all Rust types would be disastrously restrictive. Basic types like i32 or String have no self-references and can be moved safely anywhere at any time.

To resolve this, Rust introduced the trait system into the equation:

1- `Unpin`: Marker trait automatically assigned to normal, movable types (i32, String, f32). For these types, pinning is essentially a harmless transparent wrapper.

2- `!Unpin`: Marker for types that must never be relocated once pinned (such as compiler-generated async futures or types opt-in via PhantomPinned).

### SO in Breif
> Compiler-generated async futures are marked as `!Unpin`, forbidding them from moving. Here, Pin acts as a true safeguard, locking the underlying memory address in place. Similarly, custom types explicitly tagged as `!Unpin` are kept frozen.
> Conversely, standard types implement `Unpin` by default (marker traits similar to `Send` and `Sync`). Wrapping an `Unpin` type in Pin provides no functional immobility -it behaves as a simple wrapper because the type system freely permits retrieving a mutable reference to the inner value.

This operational split between `Unpin` (movable) and `!Unpin` (immovable) hinges entirely on conditional API methods like get_mut:

```rust
pub const fn get_mut(self) -> &'a mut T
where
    T: Unpin,
{/* ... */}
```

Consider that get_mut() is only available directly on Pin<&mut T> and is not directly accessible on `Pin<Box<T>>`. To access get_mut() on a pinned Box, you must first reborrow it as a pinned mutable reference using as_mut().get_mut() (provided the underlying type implements `Unpin`).

Notice the trait bound: if a type implements `Unpin`, you are allowed to safely extract a mutable reference and relocate the inner data.

While compiler-generated futures automatically opt out of `Unpin`, you can manually mark a custom struct as `!Unpin` using the `PhantomPinned` marker:

```rust
struct SelfReferring { // SelfReferring is not !Unpin
    abc: i32,
    abc_ref: *const i32,
    _pinned: PhantomPinned
}

```
Let's observe this behavior in code:

```rust
// assert if type is Unpin or !Unpin in compile time as compiler error
const fn assert_unpin<T: Unpin>(_: &T) {}
assert_unpin(&42);                                           // ✅
assert_unpin(&async { 42 });                                 // ❌
assert_unpin(&tokio::time::sleep(Duration::from_secs(3)));   // ❌ PhantomPinned
assert_unpin(&());                                           // ✅
assert_unpin(&async {                                        // ❌
    let x = 5;
    tokio::time::sleep(Duration::from_secs(3)).await;
    x
});
// ||
// || this leads to
// ||
// \/


// Unpin
let _ = Pin::get_ref(std::pin::pin!(42).into_ref());
let _ = Pin::get_mut(std::pin::pin!(42));

let _ = Pin::get_ref(std::pin::pin!(()).into_ref());
let _ = Pin::get_mut(std::pin::pin!(()));

// !Unpin
let _ = Pin::get_ref(std::pin::pin!(tokio::time::sleep(Duration::from_secs(5))).into_ref());
//let _ = Pin::get_mut(std::pin::pin!(tokio::time::sleep(Duration::from_secs(5)))); Invalid

let _ = Pin::get_ref(std::pin::pin!(async {}).into_ref());
// let _ = Pin::get_mut(std::pin::pin!(async {})); Invalid

```
A few noteworthy nuances to keep in mind:

1- You can technically bypass pinning constraints for `!Unpin` types by invoking the unsafe method `get_unchecked_mut()`, but memory safety guarantees fall entirely on your shoulders.

2- Pin itself implements `Unpin`. Moving the `Pin` container on the stack does not disrupt the memory location of the underlying pinned value.

3- If you need to transfer pinned references across thread boundaries or manage recursive futures, heap-pinning via `Pin<Box<T>>` or `Box::pin`is vastly superior to stack-pinning `(Pin<&mut T>)`.

## Final Words
Now, take another look at the signature of `poll` in the `Future` trait:

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
```
Equipped with this mental model, the API design becomes crystal clear: poll demands `Pin<&mut Self>` so that if Self is `!Unpin` (as compiler-generated async state machines are), the caller cannot inadvertently swap or relocate the state machine between poll cycles.

On the flip side, if `Self` implements `Unpin`, it gracefully falls back to standard Rust `&mut self` behavior. no pinnig, no freezing..

While pinning reaches far into the mechanics of async Rust, capturing this baseline primitive is half the battle. I hope this walkthrough gave you that much-needed "aha!" moment. Happy pinning!
