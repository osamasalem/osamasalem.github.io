+++
title = "Circus: Yet Another Programming Take in the Age of Multi-Core Processing"
date = 2025-05-06
slug = "circus-prgramming-in-multicore-era"

[taxonomies]
tags = ["rust", "programming"]

+++
Since the mid-20th century, programming languages have largely retained the same fundamental structure as when they were first invented. But as technology evolves steadily—and with hardware hitting the physical limits of single-core scaling and shifting towards multi-core architectures-there is a growing need to rethink how we approach computation.<!--more-->

Circus explores a new dimension for structuring programs in the age of multi-core systems.

## What is Circus?
`Circus` is a runtime environment and embedded programming language built around the actor model. It decomposes logic into parallel, elementary, atomic actors. Each actor receives one or more inputs and produces output in the form of workload messages passed to other actors, forming the logic graph.

For example, in a simple expression like `5 * 4 + 3 * 6`, the multiplications can be executed in parallel, with their results later combined via addition.

`Actors` are compiled into binary machine code via low-level languages and dynamically linked as modules at runtime. They are loaded on-demand (lazily) as needed.

Atomic, in this context, refers to actors that perform very small operations (e.g., arithmetic or boolean ops), and may either:

* Maintain guarded state
* Access atomic memory regions
* Be stateless altogether

Actors can be:

* **Input-only**

* **Output-only**

* **Input-output**

Messages are transmitted through high-performance, ring-buffered, LIFO queues. As actors execute, they dequeue messages, process them, and enqueue results for downstream actors. This forms a continuous processing graph.

Circus draws inspiration from Prolog (particularly in its declarative-style logic), but aims to be a general-purpose language with simpler syntax and semantics, targeting modern CPU/GPU environments.

While broadly applicable, it is especially well-suited for domains like:

- Data streaming

- Media processing

- Network pipelines

## Hello, World!
```rs
fn main():(int ret) {
    puts s("Hello, World!!");
    ret = signal_int(s.done, 0);
}
```
## Where Did the Idea Come From?
It started as an experiment in visual programming -how could software be sketched like a block diagram instead of written line-by-line?

Later, while working in networking and multithreading, this idea evolved. The goal became clearer: structure software in a way that maps naturally to parallel execution.

## Why Circus?
Several motivations support this model:

1. Legacy applications are often single-threaded, underutilizing modern CPUs.

2. The actor/message model is proven to be safe and efficient (e.g., Erlang, Go, Rust channels).

3. Replacing stacks with queues simplifies memory usage and better aligns with producer/consumer workloads and hardware caching.

## Anatomy of a Circus Program
A Circus program is a graph of connected micro-actors. Each actor (or "component") is an abstract unit with:

* Input pins (arguments)
* Output pins (return values)

Actors are instantiated and connected by logical connections between pins, creating a web of dependencies rather than a linear list of instructions.

### Runtime Behavior
At runtime, Circus:

- Spawns multiple threads (called jugglers)—typically one per core.
- Creates ring-buffered workload queues (~1.5× the number of cores).
- Each queue uses a 1MB default size (configurable).

### Jugglers:

- Pull tasks from the queues.
- Execute the associated actor logic.
- Enqueue results for other actors.
- Repeat.

### The Ringmaster
The `Ringmaster` is a special, low-priority thread that oversees the system:

* Shuts down the program when all queues are idle for a timeout period.
* Dynamically scales the number of threads or queues.
* Detects stuck threads, reassigns their workloads, and restarts them.

## Data Tags and Type Safety
The runtime is agnostic to data types. It tracks only data tags—labels defined in the script to represent semantic types. These tags:

* Must match between connected pins (strictly enforced).
* Carry no intrinsic meaning or size at runtime.

There are built-in tags like `int`, `float`, `char`, `bool`, and special signal types (e.g., void data), but they are treated symbolically. It is the actor’s responsibility to manage memory safely.

## Types of Actors
There are two categories of actors:

- **Binary actors**: Dynamically linked machine code functions.

- **Logical actors**: User-defined functions composed of internal actor graphs.

### Example of Logical Actor:
```rs
fn foo(int a, int b):(int ret) {
    int c = a + b;
    string str = (string)c;
    puts(str);
    ret = 1234;
}
```

**Note**: The assignment to ret is independent of other operations.

## No Recursion, But Repetition Allowed
Due to its graph-based structure, recursion is not possible-actors cannot reference themselves cyclically.

However, loops (repetition) can be modeled via feedback cycles in the graph:

```rs
x = 0;
x = x + 1;
```
This creates an infinite loop where `x` is a cache actor holding the latest value. The value is incremented and re-fed into `x`.

## Data Handling
By default, data is copied into workloads. Avoid using raw pointers unless dealing with large shared structures.

To share safely:

* Use copy-on-write memory with reference counting.

* Store shared state in dedicated, guarded actors.

## Avoid Recomputing Unchanged Data
To optimize, Circus propagates only changed data.

Each value has a 64-bit hash, so if an actor receives the same value again, it can skip reprocessing.

```rs
x = 0;
x = x * 1;
```
This will terminate, since the hash of `x * 1` remains unchanged.

## Sequence Numbers (Optional Feature)
Circus could also support global sequence numbers for each value, enabling:

* Detection of **outdated values**.
* Preference for newer inputs (as in networking with packet ordering).

## Dual Nature of Actors
Actors can be called like functions or used like objects:
```rs
fn foo(string s, float f):()

// Functional call
foo("abc", 1.2);

// Object-like usage
foo f;
f.s = "abc";
f.f = 1.2;

// Constructor-style
foo f("abc", 1.2);
```

## Join and Fork
Two special actor types:

**Join**: Two inputs, one output. Emits whenever either input is updated.

**Fork**: One input, two outputs. Duplicates input to both outputs.

## Future Horizons
**Hardware support**: Future processors might embrace models like Circus natively.

**Multi-tenant distribution**: Sharing workloads across machines for fault tolerance and scalability.

**Adaptive resource control**: Runtime could self-regulate based on real-time load using control theory principles.
