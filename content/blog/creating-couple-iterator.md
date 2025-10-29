+++
title = "Creating couple iterator: Another dive into unsafe Rust!!"
date = 2025-10-29
slug = "creating-couple-iterator"

[taxonomies]
tags = ["programming","rust"]

+++
I was working on an N-body simulation in Rust, and I noticed a gap:
Rust, out of the box, does not provide an iterator that can safely yield pairs of mutable references (&mut T) from a slice - without violating Rust’s borrowing rules.

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

However, we know that getting mutable references to different elements is perfectly safe - their memory regions don’t overlap. So, unsafe code is the only way forward.

## Implementation
> ⚠️ DISCLAIMER: This code is for demonstration purposes only. It is not optimized or production-ready, but it highlights the concept clearly.

Let’s start with the iterator structure.
A standard Rust slice iterator internally uses a `NonNull<T>` pointer to track the current element and walks until it reaches the end pointer.

Here, we’ll maintain two pointers - one for each element of the pair:

```rust
#[derive(Clone)]
struct CouplesIterator<'a, T> {
    first: NonNull<T>, // the first element in the pair tuple
    second: NonNull<T>, // the second element in the pair tuple
    end: *mut T, // the end pointer
    _dummy: PhantomData<&mut 'a T>, // a friend to cover the presence of 'a
}
```
The logic is simple:

- `first` starts at the first element,
- `second` starts one step ahead,
- `second` advances until it reaches the end,

when it does, first advances by one, and second resets to one step ahead of it.

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
This works fine for types with a nonzero size (e.g., i32, f64, etc.), but what happens if the type has zero size - like the unit type () or an empty struct?

consider this
```rust
struct ZeroSizedTypes;W
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
(Interestingly, the Rust standard library optimizes this internally - since `*mut T` and `usize` have the same size and alignment, it reuses the same field for both cases.)

```rust
if mem::size_of::<T>() == 0 {
    /* Zero size */
} else {
    /* Non zero size */
}
```
You might wonder:

> “If both pointers refer to the same object, isn’t it UB to create two mutable references to it?”

Normally yes- except for ZSTs. Since ZSTs occupy no memory, aliasing doesn’t lead to any data overlap or race conditions. as they are all ending up being no-ops.

So, we can safely create multiple &mut references to the same ZST value.

```rust
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

            let (second, end) = if mem::size_of::<T>() == 0 {
                let count = len * len.wrapping_sub(1) /2;
                (first, IterationMode::Count(count))
            } else if len < 2 {
                (first, IterationMode::Count(0))
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

I made a few assumptions and simplifications for clarity, so if you spot something worth improving, please reach out or comment - I’d love to hear your thoughts!
