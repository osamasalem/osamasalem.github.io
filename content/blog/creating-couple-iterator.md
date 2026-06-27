+++
title = "Creating couple iterator: Another dive into unsafe Rust!!"
date = 2025-10-29
slug = "creating-couple-iterator"

[taxonomies]
tags = ["programming","rust"]

+++
I was working on an N-body simulation in Rust, and I noticed a gap:
Rust, out of the box, does not provide an iterator that can safely yield pairs of mutable references (`&mut T`) from a slice - without violating Rust’s borrowing rules.

So, I decided to implement one myself and dive once again into the world of unsafe Rust.
<!--more-->

In my simulation, I needed a way to handle pairwise interactions - forces and collisions between two bodies - and update their positions and velocities through mutable references.
This pattern is quite common in game development and physics simulations.

Imagine we have:
```rust
let myvec = vec![1, 2, 3, 4];
```
We want an iterator that produces pairs of distinct mutable references:

```rust
(&mut 1,&mut 2)
(&mut 1,&mut 3)
(&mut 1,&mut 4)
(&mut 2,&mut 3)
(&mut 2,&mut 4)
(&mut 3,&mut 4)
```

In safe Rust, this is impossible- you can’t create two mutable references into the same slice at once without splitting it first.
The borrow checker prevents this to ensure aliasing safety.

However, we know that getting mutable references to different elements is perfectly safe - their memory regions don’t overlap. So, unsafe code is the way to go.

> You can -also- write this iterator if you split the array, but you have to add indeces and caching.. etc, Look! I chose to use unsafe !!

## Implementation
> ⚠️ DISCLAIMER: This code is for demonstration purposes only. It is not optimized or production-ready, but it highlights the concept clearly.

Let’s start with the iterator structure.
A standard Rust slice iterator internally uses a `NonNull<T>` pointer to track the current element and walks until it reaches the end pointer.

> [NonNull](https://doc.rust-lang.org/std/ptr/struct.NonNull.html) is **zero-cost-abstraction** new-type for **non-null pointers** (Pointers that impossible to be NULL)

Here, we’ll maintain two pointers - one for each element of the pair:

```rust
#[derive(Clone)]
struct CouplesIterator<'a, T> {
    first: NonNull<T>,              // the first element in the pair tuple
    second: NonNull<T>,             // the second element in the pair tuple
    end: *mut T,                    // the end of slice pointer
    _dummy: PhantomData<&mut 'a T>, // to tie the lifetime 'a
}
```
> [PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html): is a **zero-sized placeholder** for **generic types** and **lifetimes** that are not involved in structs' data fileds (which probably used in implementations), whithout it, the compiler will complain that the generic/lifetime parameter is never used.

The logic is simple:

`first` starts at the first element, `second` starts one step ahead,`second` advances until it reaches the end, when it does, `first` advances by one, and second resets to one step ahead of it.

If the slice has fewer than two elements, we stop immediately.

Here’s a simplified version of `next()`:

```rust
impl<'a, T: 'a> Iterator for CouplesIterator<'a, T> {
    type Item = (&'a mut T, &'a mut T);
    fn next(&mut self) -> Option<Self::Item> {

        unsafe {
            //...

            if self.second.as_ptr() == end {
                self.first = self.first.add(1);
                self.second = self.first.add(1);
            }

            if self.first.as_ptr() == end.sub(1) {
                return None;
            }

            let first = self.first.as_mut();
            let second = self.second.as_mut();

            self.second = self.second.add(1);

            Some((first, second))
        }
    }
}

```

## ZST, Whooow!!! 👻
This works fine for types with a nonzero size (e.g., `i32`, `f64`, etc.), but what happens if the type has zero size - like the unit type () or an empty struct?

consider this
```rust
struct ZeroSizedTypes;
```

For ZSTs, all instances have the same address (Rust guarantees they’re logically distinct but occupy no memory).
This means our pointer-based approach breaks down- first, second, and end all point to the same location!

To handle this safely, we can fall back to an iteration counter instead of pointer arithmetic.The total number of pairs is `n * (n - 1) / 2`, where `n` is the slice length.

We can encode both cases (ZST and non-ZST) using an enum:
```rust
enum IterationMode<T> {
    Count(usize),
    Pointer(*mut T),
}
```
Interestingly, the Rust standard library optimizes this internally - since `*mut T` and `usize` have the same size and alignment, it reuses the same field for both cases.

```rust
if mem::size_of::<T>() == 0 {
    /* Zero size: end as usize */
} else {
    /* Non zero size: end as pointer */
}
```

## Safety Note on ZST Aliasing

You might wonder:

> “If all pointers refer to the same object, isn’t it UB to create two mutable references from one mutable pointer?”

Normally yes- except for ZSTs. Since ZSTs occupy no memory, aliasing doesn’t lead to any data overlap or race conditions. as they are all ending up being no-ops.

So, we can safely create multiple `&mut` references to the same ZST value.

```rust
use std::*;
use std::mem::*;
use std::ptr::NonNull;
use std::marker::PhantomData;

trait SliceEx<'a, T>
where
    T: Sized,
{
    fn couples_mut(self) -> CouplesIterator<'a, T>;
}

impl<'a, T> SliceEx<'a, T> for &'a mut [T] {
    fn couples_mut(self) -> CouplesIterator<'a, T> {
        unsafe {
            let len = self.len();

            let first = NonNull::from_mut(self).cast();

            let (second, end) = if mem::size_of::<T>() == 0 || len < 2 {
                let count = len * len.wrapping_sub(1) /2;
                (first, IterationMode::Count(count))
            } else {
                (
                    first.add(1),
                    IterationMode::Pointer(first.add(len).as_ptr()),
                )
            };

            CouplesIterator {
                first,
                second,
                end,
                _dummy: PhantomData,
            }
        }
    }
}


#[derive(Clone)]
enum IterationMode<T> {
    Count(usize),
    Pointer(*mut T),
}


#[derive(Clone)]
struct CouplesIterator<'a, T> {
    first: NonNull<T>,
    second: NonNull<T>,
    end: IterationMode<T>,
    _dummy: PhantomData<&'a mut T>,
}

impl<'a, T> Iterator for CouplesIterator<'a, T> {
    type Item = (&'a mut T, &'a mut T);
    fn next(&mut self) -> Option<Self::Item> {
        unsafe {
            match self.end {
                IterationMode::Count(ref mut count) => {
                    if *count == 0 {
                        return None;
                    }
                    *count -= 1;
                    let first = self.first.as_mut();
                    let second = self.second.as_mut();
                    Some((first, second))
                }
                IterationMode::Pointer(end) => {
                    if self.second.as_ptr() == end {
                        self.first = self.first.add(1);
                        self.second = self.first.add(1);
                    }

                    if self.first.as_ptr() == end.sub(1) {
                        return None;
                    }

                    let first = self.first.as_mut();
                    let second = self.second.as_mut();

                    self.second = self.second.add(1);

                    Some((first, second))
                }
            }
        }
    }
}

#[derive(Debug, Clone)]
struct S;


fn main() {
    let k: &mut [i32] = &mut [1, 2, 3, 4];
    k.couples_mut().for_each(|(a, b)| mem::swap(a, b));
}

```
## Final words

I really enjoyed this deep dive - tinkering at a low level in Rust is always fun and enlightening.
This experiment revealed interesting edge cases, especially around ZST aliasing and iterator design in unsafe Rust.

Handling ZSTs is also discussed in the [Rustonomicon](https://doc.rust-lang.org/nomicon/vec/vec-zsts.html) which takes a different approach to dealing with them. I personally find that method a bit more of a gimmicky - but it’s definitely worth checking out if you’re curious.

I made a few assumptions and simplifications for clarity, so if you spot something worth improving, please reach out or comment - I’d love to hear your thoughts!

**UPDATE 27/Jun/2026**:
After thinking about this implementation for a while, I realized that although it appears to work correctly in a typical for loop, it is actually unsound.

The problem is that Iterator::next returns references with the lifetime 'a, allowing callers to keep previously returned items alive while calling next() again. This can create multiple mutable references to the same element

consider this example
```rust
let mut v = vec![1,2,3,4];
let mut it = v.couples_mut();
let first = it.next(); // (&mut 1,&mut 2);
let second = it.next(); // (&mut 1,&mut 3); // WRONG; we have multiple mut references
                        // pointing to the same 1 element

```

At this point, both `first.0` and `second.0` are distinct `&mut` references to the same element `(v[0])`. This violates Rust's aliasing guarantees, making the iterator unsound and causing undefined behavior-even though no unsafe code is used by the caller.

A for loop usually doesn't expose this issue because each yielded item is dropped before the next iteration. However, the Iterator trait itself makes no such guarantee. Callers are free to keep multiple items alive simultaneously, so an implementation must be sound for all valid uses of the trait, not just the common ones.

A better design would be to return a single mutable reference together with an iterator over the remaining elements. Conceptually, the API would look something like:

```rust
for (current, rest) in v.couples_mut() {
    for other in rest {
        // (current, other)
    }
}
```
This design naturally expresses the invariant that current and rest are disjoint, allowing the implementation to remain entirely safe.

If I have some more time, I'd like to experiment with this approach and see how ergonomic it feels in practice.
