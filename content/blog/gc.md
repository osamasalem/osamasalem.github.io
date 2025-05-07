+++
title = "Criticizing Automated Garbage Collection"
date = 2025-05-07
slug = "rethinking-gc"

[taxonomies]
tags = ["programming","rust"]

+++




Garbage collection (GC) is widely considered a hallmark of modern programming languages, promoted for its ability to simplify development and reduce memory-related bugs. Its presence in languages such as Java, C#, Go, and JavaScript has become ubiquitous.

However, <!--more--> while GC may reduce development complexity, it introduces non-trivial runtime costs—costs that are ultimately borne by the end user. This article presents a critique of GC from a user-experience and engineering accountability perspective, advocating for deterministic memory management models where appropriate, especially in production software.

## 1. The Ethical Dimension: Who Pays the Cost?
From the perspective of an end user, runtime performance matters. Whether on mobile, desktop, or embedded systems, users value responsiveness, energy efficiency, and predictability. Yet garbage collection introduces behavior that can contradict these values:

* Unpredictable pauses: Even with concurrent or incremental GC strategies, stop-the-world events can still occur, causing latency or frame drops in UI or real-time systems.

* Increased memory footprint: GC-managed systems often over-allocate memory to reduce collection frequency, raising the baseline resource usage.

* Indirect power consumption: On battery-powered devices, background memory sweeps have energy implications.

These trade-offs benefit the developer, not the user. The question arises: Is it justifiable for user-facing software to offload developer convenience to the runtime at the user’s expense?

## 2. GC is a Solution to a Developer Problem
Garbage collection addresses issues of memory safety and object lifecycle management, traditionally seen as burdensome or error-prone. While these are legitimate concerns, they are developer-side concerns—and thus, should be solved during development, not deferred to runtime.

Modern alternatives like RAII (Resource Acquisition Is Initialization) in C++ and Rust’s ownership system provide deterministic, compile-time memory safety without runtime garbage collection. These models shift the responsibility where it belongs: onto the developer and the compiler, not the end user.

Furthermore, the emergence of linear and affine type systems, particularly in experimental and research-oriented languages, shows promise for compile-time enforcement of resource constraints without any runtime penalty.

## 3. Auxiliary Runtime Services: A False Equivalence
It’s reasonable to observe that most production applications ship with runtime services such as:

* Logging and telemetry

* Tracing and auditing tools

* Monitoring agents

However, these services are typically optional, externally configurable, and dynamically throttled or disabled in production builds or constrained environments. By contrast, GC is non-optional in GC-based languages and intrinsically tied to program execution, making it much harder to isolate or mitigate.

## 4. The Android vs iOS Case Study
A practical illustration of GC’s drawbacks can be seen in the historical performance disparity between Android and iOS:

* Android applications, primarily written in Java/Kotlin, rely on garbage-collected runtimes like Dalvik or ART.

* iOS applications, built in Objective-C and Swift, rely on reference counting with ARC (Automatic Reference Counting), providing deterministic release behavior.

While hardware differences and OS-level integration play roles, developers and analysts have often cited GC-related pauses and memory usage patterns as contributing to Android's historic performance and battery issues—especially on lower-end devices.

## 5. Capital vs Operational Cost Trade-off
Proponents of GC argue that it reduces time-to-market and developer effort, which translates to lower capital expenses. However, GC can increase operational expenses, especially at scale:

* More CPU cycles consumed per server

* Higher memory pressure across replicated services

* Increased cost for hosting, monitoring, and tuning long-lived applications

For software deployed at scale or on constrained edge devices, this trade-off may not be justified.

## 6. A More Nuanced Position
While GC simplifies development in early-stage projects, proofs of concept, or command-line tools with brief execution windows, its inclusion in long-running, user-facing software should be deliberate, measured, and justifiable.

> Garbage collection is a practical engineering shortcut — not a universally valid foundation for software design.

In contexts where performance, determinism, and control are essential, systems with compile-time memory guarantees—like Rust, modern C++, or languages with linear types—are not just preferable, but ethically aligned with delivering the best possible experience to users.

## Conclusion
Garbage collection solves real problems, but it is not a silver bullet. Developers should recognize it as one trade-off among many—not as a default. As an industry, we should strive for tools and languages that encode correctness and efficiency at compile time, rather than relying on invisible background processes at runtime.

Our goal should be clear: to deliver robust software that respects user resources, and to take full ownership of the systems we build—including their memory lifecycles.

**By Osama Salem (and ChatGPT as technical sounding board)**
