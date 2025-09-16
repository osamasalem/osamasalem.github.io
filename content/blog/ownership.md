+++
title="Moving the unmovable: how to transfer ownership in Rust"
date=2025-08-22
slug="moving-the-unmovable"
[taxonomies]
tags = ["programming","rust"]
+++
Good software design often requires carefully modeling ownership and data flow from the start. In Rust, this is especially important because ownership is at the core of the language. Sometimes, though, you need to move ownership of an object around- and for many developers coming from other languages, this feels unfamiliar and even frustrating.

This post discusses techniques for moving owned objects in Rust.<!--more--> By default, Rust prevents you from simply “grabbing” an object out of another, because doing so might break invariants and leave objects in an invalid state.

Rust’s ownership model is built on two fundamental rules:

- No object may ever exist in an invalid state.
- Every value has exactly one owner.

From these rules, it follows that moving a child object out of its parent is normally prohibited, since it could invalidate the parent. In other words, Rust enforces a solid ownership tree: once objects are instantiated, their structure usually remains fixed.

> Note: We’re talking here about owned values, not primitives or Copy types.

To explore the options, we’ll use a computer setup as our example: a computer must have a monitor. Throughout the post we’ll see how to “move the monitor out of the setup” under different circumstances.

But first, Let's see the problem, and illustrate how Rust refuses to move objects around
```rust
struct Child;
struct Parent {
    child: Child,
}

fn lets_move_the_unmovable(parent: Parent) -> (Parent, Child) {
    let child = parent.child;
    (parent, child)
}
```

this code will cause this comiler error
```
error[E0382]: use of partially moved value: `parent`
 --> src\main.rs:8:6
  |
7 |     let child = parent.child;
  |                 ------------ value partially moved here
8 |     (parent, child)
  |      ^^^^^^ value used here after partial move
  |
  = note: partial move occurs because `parent.child` has type `Child`, which does not implement the `Copy` trait

```

## 0. Other means

Before trying to move ownership directly, consider alternatives:

- References: Shared (`&T`) or exclusive (`&mut T`) references are often enough.
- Shared ownership: If multiple objects need access, use `Arc<T>` or `Arc<RefCell<T>>`.
- Cloning: `.clone()` gives you a copy (sometimes expensive, but valid). This doesn’t move the original, but in some cases that’s exactly what you want.

## 1. Break the whole setup to get the child

If you really want to extract a child, the simplest way is to destroy the parent.
For example: to get the monitor, throw away the whole computer setup. simple like that!!
consider this snippet

```rust
struct Child;
struct Parent {
    child: Child,
}

fn drop_parent_take_child(mut parent: Parent) -> ((), Child) {
    let child = parent.child; // this is allowed as the funtion takes the ownership of
                              // the parent will be destructed at the end of this function
    // theoritical drop(parent)
    ((/*No parent*/), child)
}
```

This works, but if you had a deep ownership tree, you’d have to destroy the entire structure just to reach a leaf value- sometimes impractical.

## 2. Barter with the owner

What if you want the monitor, but don’t want to harm the computer? You can replace the monitor with similar one.

check this

```rust
fn replace_with_new_one(mut parent: Parent) -> (Parent, Child) {
    // take the object
    let child = parent.child;
    // compensate the owner
    parent.child = Child;
    (parent, child)
}
```

though the idiomatic way may be something like this:

```rust
fn swap(mut parent: Parent) -> (Parent, Child) {
    let mut child = Child;
    // barter objects
    mem::swap(&mut child, &mut parent.child);
    (parent, child)
}
```

If `Child` struct implements `Default` trait, you can use this API call to replace the child object with the default, something like this

```rust
#[derive(Default)]
struct Child;

fn replace_with_default(mut parent: Parent) -> (Parent, Child) {
    // hey owner!! I will take this and you can use the default
    let child = mem::take(&mut parent.child);
    (parent, child)
}
```

Note: Usually, you `clone` the objects in Rust which is similar to bartering. but instead we get the clone out from the owned value, which we can not consider it as transfer ownership.

```rust
#[derive(Clone)]
struct Child;

fn replace_with_default(parent: Parent) -> (Parent, Child) {
    (parent, parent.child.clone())
}
```

## 3. Replace it with a placeholder

Another approach is to make the parent more flexible. Instead of requiring a child, allow for the absence of one with Option.

consider this one:

```rust
struct TolerantParent {
    child: Option<Child>,
}

fn take_and_fill_the_gap(mut parent: TolerantParent) -> (TolerantParent, Option<Child>) {
    if let child = parent.child.take();
    (parent, child)
}
```

Now, the parent remains valid even if the child is missing. The cost is that you’ve moved some safety checks from compile time to runtime- None means “the child is gone.”

## 4. In containers things can be easier

With collections, Rust often provides convenient APIs. For example, `Vec::remove` gives you ownership of an element while maintaining the rest:

```rust
fn take_one_of_vec(parent: Vec<Child>) -> (Vec<Child>, Child) {
    let mut parent = parent;
    let child = parent.remove(0);
    (parent, child)
}
```

If your computer setup has multiple monitors, taking one doesn’t invalidate the rest.

But pay attention, that this may obey the general algorith call complexity cost. so if you are okay with that, no problem !!

## Final Thoughts

We’ve walked through several techniques for transferring ownership in Rust. The language forces us to preserve invariants and prevent invalid states- but within those rules, there are multiple strategies to move values around safely.

Even if none of these cases match your exact problem, I hope this gave you a clearer sense of how Rust’s ownership works. Keeping invariants valid at all times is exactly what makes Rust special in resource management.
