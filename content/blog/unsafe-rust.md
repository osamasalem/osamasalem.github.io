+++
title="Unsafe Rust"
date=2025-05-12
slug="unsafe-rust"
[taxonomies]
tags = ["programming","rust"]
+++


Back in college, I overheard a group of students discussing something interesting: C#, Microsoft's proposed alternative to C++ and their own dialect of Java. I asked, “What about accessing low-level data types and structures?” Someone responded, “You can use `unsafe` blocks to access pointers freely and interoperate with the newly introduced .NET CLR.” The CLR, of course, allows you to write code in VB, C#, or even C++.
<!--more-->
The `unsafe` feature in C# was intended to grant access to pointers while isolating such operations from the rest of the safe program.

C# is considered one of the *memory-safe languages*, despite the presence of `unsafe`. No one seriously questioned its safety model just because of this feature. That’s because safe languages don't exist in a vacuum. In the real world, we often have to bridge the gap between safe environments and the messy, unsafe outer world.

And it's not just C#. Java has JNI and `Unsafe` classes. Elixir has NIFs. Python has Cython and C extensions. These don’t compromise the core safety model of the language—they complement it.

Rust unsafe keyword is a pragmatic escape hatch to interact with the outer world, without switching the languages like in cases of managed/scripting languages (Go/Java/Python),

In theory, in the Rust utopia—where Rust angels flutter beside unicorns under everlasting rainbows, running software on formally verified operating systems over safe hardware instruction sets—`unsafe` shouldn’t exist.

But as this is not the case for the world, it will not be the case for Rust !

## What is unsafe Rust.
Let’s begin with a simple but essential concept. Consider the following code:

```rs
///# Safety
/// This will summon a demon from hell
unsafe fn summon_demon() -> Demon {
    Demon(666)
}

fn main() {
    unsafe {
        let satan = summon_demon();
    }
}

```

Looks like a normal function, right? It is. But it’s marked as unsafe because the implementers determined it could lead to undefined or erroneous behavior if misused. (And maybe some biblical implications, depending on your architecture. 😅)

Here, the unsafe keyword acts as a marker. It signals to developers and reviewers that this code deserves extra scrutiny. Think of it as code highlighting—literal function coloring.

What’s critical to understand is that the kinds of issues unsafe code can introduce are fundamentally unpresentable in Rust’s type system or runtime checks. It’s not that the compiler or runtime is neglecting to help—it’s that these operations lie beyond what can be statically verified or dynamically enforced. That’s why they’re marked unsafe: not because they are necessarily dangerous, but because their safety cannot be proven by the language.

Notice the doc comment above the function? It's not enforced by the compiler, but linters strongly recommend adding such safety notes to describe any preconditions, assumptions, or hazards relevant to the unsafe code.

In my opinion, the word unsafe might even be misleading. Maybe terms like unverifiable or unfalsifiable are more accurate. Because what’s marked unsafe might be perfectly safe in many contexts—it's just that the compiler can't guarantee it.

## what Unsafe does that really mean

As we said, unsafe is just a marker—it doesn’t turn off Rust’s safety checks. It simply unlocks certain extra operations. According to [Rust book](https://doc.rust-lang.org/book/ch20-01-unsafe-rust.html#unsafe-superpowers) these include:

- Dereference a raw pointer
- Call an unsafe function or method
- Access or modify a mutable static variable
- Implement an unsafe trait
- Access fields of a `union`

It’s important to emphasize:
> All of Rust’s safety guarantees remain intact inside unsafe blocks—unless you explicitly use these operations.

```rs
fn main() {
    unsafe {
        let hello = String::from("hello");
        let another_hello = hello;
        println!("{hello}");
    }
}

```
This code will raise this error
```
error[E0382]: borrow of moved value: `hello`
  --> src\main.rs:10:19
   |
8  |         let hello = String::from("hello");
   |             ----- move occurs because `hello` has type `String`, which does not implement the `Copy` trait
9  |         let another_hello = hello;
   |                             ----- value moved here
10 |         println!("{hello}");
   |                   ^^^^^^^ value borrowed here after move
   |
   = note: this error originates in the macro `$crate::format_args_nl` which comes from the expansion of the macro `println` (in Nightly builds, run with -Z macro-backtrace for more info)
help: consider cloning the value if the performance cost is acceptable
   |
9  |         let another_hello = hello.clone();
   |                                  ++++++++

```
But wait—we’re in an unsafe block, right? Yes, but safety checks still apply unless you’re doing one of the explicitly unsafe operations listed above.

In fact, every unsafe operation in Rust typically has a safe counterpart: pointers correspond to references, mutable statics to RefCells, etc.

## But Why ?

As mentioned, unsafe is a pragmatic escape hatch for rare, specific scenarios. I’ll borrow (with respect!) from [Jon Gjengset's charming speech from Rust NYC meet up 2019](https://youtu.be/QAz-maaH0KM?t=83) but rephrase in my own words:

- Interfacing with non-Rust ecosystems (e.g., C libraries/ OS syscalls/ Controllers memory or interrupts)
- Building performance-critical abstractions that can’t be expressed in safe Rust

Even in these cases, we encapsulate the unsafe code inside safe abstractions. It’s like wiring electricity behind the walls—unsafe at the core, but safe at the surface.

From experience: don’t jump to unsafe immediately. Often, there is a safe Rust way to achieve what you need, even if it’s not well-known. I’ve personally rushed into unsafe, only to later discover a beautiful, idiomatic solution in safe Rust.



## Final Thoughts

One thing that puzzled me: once Rust gained traction, people—often with differing motives—started questioning the very definition of memory safety. What is memory safety, anyway? How do we distinguish between memory-safe languages and memory-safe outputs?

While healthy debate is important, sometimes it veers into philosophical overkill. We end up asking “What is the Sun? What is the Moon? What are trees?”—as if software engineering were invented yesterday.

Memory-safe languages are not a new idea. What Rust did was shift the cost of enforcing safety from post-production (runtime crashes, CVEs, etc.) to pre-production (compile-time errors). That’s not perfection—but it’s a major leap forward.

What Rust aims to achieve is the ability to write a vast sea of everyday, traditionally tasks like database access, file I/O, and network communication—safely, without touching unsafe at all. And when you truly need low-level access—say, to the operating system, hardware, or specialized performance-critical code—you isolate that logic into small islands of unsafe. It’s a model that preserves safety where it’s feasible, and gives you control only where it's absolutely necessary. Elegant, isn’t it?

