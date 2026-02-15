+++
title="Windows crate: The good, the bad and the awkward."
date=2026-02-13
slug="windows-crate"
[taxonomies]
tags = ["programming","rust"]
+++
The [windows crate](https://crates.io/crates/windows) is the Rust package from Microsoft that provides Rust-friendly bindings to the Windows API, which is essential for any kind of serious system programming on Windows.
<!--more-->

I’ve used this package in a couple of recent projects, working with both Microsoft provided crates: `windows` and `windows-sys`. This post focuses on my experience and thoughts about the former.

## The good

The API coverage is honestly impressive (much more than I initially expected). The granularity of what’s exposed is mindblowing: even very small ABI-level details are covered. That alone deserves appreciation.

Another big win is error handling. Wrapping Windows error codes inside Rust’s `Result<T, E>`, with automatic error message translation, is a huge quality-of-life improvement. Anyone who has wrestled with [FormatMessageW](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-formatmessagew) knows how painful that can be, so this abstraction is something I genuinely love.

COM handling is also well done. Reference counting via RAII fits naturally into Rust’s ownership model, and pairing it with explicit Clone semantics and automatic Drop feels correct and intuitive. Once you internalize it, there’s very little mental overhead.

## The bad

Some Windows APIs are inline functions (i.e. [MFGetAttributeSize](https://learn.microsoft.com/en-us/windows/win32/api/mfapi/nf-mfapi-mfgetattributesize) they’re part of the Windows API surface but not part of the ABI). These functions are documented on Microsoft Learn, but they don’t exist as exported symbols, so they’re understandably missing from the windows crate.

Porting them manually isn’t difficult, but it is a bit annoying, especially when the documentation exists and the function feels “official.”

Another mild annoyance is the feature-gating model. Mapping APIs to crate features using the [official metadata site](https://microsoft.github.io/windows-rs/features/) is actually quite well done, the site is simple and effective,but needing to add a Cargo feature just to call a single API can feel gimmicky.

I’m not sure what the perfect solution is here. Maybe some commonly used APIs could be included in more accessible `default` feature sets,but that’s easier said than done.

It would also be nice if more of the Microsoft API documentation were surfaced directly in `docs.rs`. This is a minor complaint, but it would improve discoverability and ergonomics.

As in the good side of COM and RAII of windows crate, There are some bad faces where allocated array of COM objects are not handled idiomatically in RAII, So it is worth-mentioning that you have to handle it yourself using something like [CoTaskMemFree](https://learn.microsoft.com/en-us/windows/win32/api/combaseapi/nf-combaseapi-cotaskmemfree).

## The awkward

Some Windows APIs really cry out for idiomatic Rust equivalents.

In the process of preserving Win32 API compatibility and layering Rust safety on top, you sometimes end up with signatures that look… strange:
```rust
pub unsafe fn MFEnumDeviceSources<P0>(
    pattributes: P0,
    pppsourceactivate: *mut *mut Option<IMFActivate>, // Wha..?
    pcsourceactivate: *mut u32,
) -> Result<()>
where
    P0: Param<IMFAttributes>;

pub unsafe fn ReadSample(
    &self,
    dwstreamindex: u32,
    dwcontrolflags: u32,
    pdwactualstreamindex: Option<*mut u32>,
    pdwstreamflags: Option<*mut u32>,
    plltimestamp: Option<*mut i64>,
    ppsample: Option<*mut Option<IMFSample>>, // Huh !!
) -> Result<()>;
```

I understand why these look the way they do. They’re mechanically faithful to the original API, and if you already know the Win32 conventions, they’re workable.

But for beginners or even experienced Rust developers new to Windows APIs this can be genuinely puzzling.

Once you hit APIs that return arrays of COM objects via out parameters, the abstraction starts to unravel a bit, and the mental model becomes harder to maintain.

Because of this, I often find myself leaning toward `windows-sys` instead. It’s much closer to the raw API and much farther from idiomatic Rust, but that’s a personal preference, not a universal judgment.

## Final thoughts
I’m keeping an eye on the [winsafe](https://crates.io/crates/winsafe) crate. Its coverage is steadily improving, and its API is more idiomatic, but it’s still relatively immature and missing large portions of the Windows surface area.

Overall, my experience with the windows crate has been straightforward and stable. Development was reliable. and while there are some bumps and quirks, none of them were deal-breakers.

If you’re doing serious Windows system programming in Rust today, windows is still the most complete and practical option available.
