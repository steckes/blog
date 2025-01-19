---
title: "Bringing RealtimeSanitizer to Rust"
date: "2023-11-12"
summary: "A guide to integrating LLVM's RealtimeSanitizer in Rust applications."
description: "Learn how to detect and prevent real-time violations in Rust using RTSan, a powerful tool for real-time programming."
toc: false
readTime: true
autonumber: false
math: false
tags: ["rust", "rtsan"]
showTags: false
hideBackToTop: false
---

A few months ago, I gave my first talk about Rust for audio apps at [AudioDevCon](https://audio.dev/) in Bristol. There, I met Chris and David, who presented their work on [RealtimeSanitizer (RTSan)](https://clang.llvm.org/docs/RealtimeSanitizer.html) - a tool that will be officially included in LLVM20. RTSan is a powerful addition to the developer toolkit, helping detect real-time violations in your code that can be particularly challenging to spot, especially for newcomers to real-time programming.

At the time, RTSan was only available for C++, and they were looking for someone to help make it accessible from Rust, given the growing interest in the community. Since this tool would be valuable for my daily work, I saw it as an opportunity to learn and create my first serious open source library. I took on the challenge, and now the first version is available!

> **Source Code:** [https://github.com/realtime-sanitizer/rtsan-standalone-rs](https://github.com/realtime-sanitizer/rtsan-standalone-rs)

## Understanding Real-time Constraints

Real-time programming requires guaranteeing that code will complete within a specific time frame. Take self-driving cars as an example: when detecting a person crossing the street, the car must apply the brakes within a guaranteed number of milliseconds. Even a one-in-a-million timing failure would be unacceptable.

In audio programming, while not as critical as automotive safety, we still face real-time requirements. Missing our timing window results in audible glitches - at best, it's annoying for the user, at worst, imagine your code is running in the audio chain at a major concert and an audio glitch blasts through the PA system, potentially damaging expensive equipment and causing hearing damage.

Real-time requirements are also crucial in aerospace, robotics, industrial automation, medical devices, game development, and high-frequency trading systems where microsecond-level precision can make the difference between profit and loss.

## What Makes Code Non-Real-time Safe?

Real-time safe code requires predictable execution time for every line of code. You can't block execution waiting for operations outside your control. This rules out network requests, file I/O, thread creation, memory allocation/deallocation, and often mutex locking and unlocking.

While these constraints might seem manageable when working with basic language features, even standard library functions can cause headaches - leading to lengthy discussions and conference talks. Beyond the standard library, using third-party code becomes risky without thorough code review, as you can't be certain what happens under the hood of each function call.

## How RTSan Works

This is where RTSan comes in. It raises runtime errors when detecting real-time violations like malloc, free, pthread_mutex_lock, and system calls. While not perfect, it's an invaluable tool. I've already had several eye-opening moments discovering memory allocations in places I thought were allocation-free.

## Making it Usable from Rust

RTSan will be available with LLVM20, which Rust will adopt eventually. It could then be integrated like other sanitizers in **unstable** Rust and used with the compiler flag `-Zsanitizer=realtime`.

Unlike existing sanitizers like AddressSanitizer, which typically check entire programs, RTSan needs to target specific real-time sections of your code. Non-real-time parts, like initialization, don't need these checks. That's why RTSan introduced the `[[clang::nonblocking]]` attribute to activate the sanitizer for specific functions, and `[[clang::blocking]]` to mark known non-real-time-safe functions that the sanitizer might miss.

While these attributes could be added to unstable Rust (similar to the current [no_sanitize](https://doc.rust-lang.org/unstable-book/language-features/no-sanitize.html) flag), we wanted to provide immediate access in stable Rust. This led to the creation of [rtsan-standalone-rs](https://github.com/realtime-sanitizer/rtsan-standalone-rs) - "standalone" distinguishing it from the future built-in version.

## The Implementation

The implementation was straightforward thanks to Dave and Chris's existing [C-Header](https://github.com/realtime-sanitizer/rtsan/blob/main/include/rtsan_standalone/rtsan_standalone.h), which provides runtime functions to control the sanitizer.

The Rust wrapper consists of three crates:

- **rtsan-standalone**: The main crate that provides safe wrappers for the C functions and defines the `scoped_disabler` macro for temporarily disabling sanitizer checks.

- **rtsan-standalone-macros**: Implements the procedural macros `nonblocking`, `blocking`, and `no_sanitize_realtime` - these are the primary interfaces you'll use when working with RTSan.

- **rtsan-standalone-sys**: Handles the low-level integration by generating bindings to the C-Header, downloading LLVM20 pre-release source files, and building the RTSan library. The build process can take several minutes, producing a dynamic library on macOS and a static library on Linux. To avoid rebuilding, you can copy the RTSan library to a persistent location and set the `RTSAN_LIBRARY_PATH` environment variable.

## Using the Sanitizer

The sanitizer is opt-in via a feature flag. Add this to your Cargo.toml:

```toml
[dependencies]
rtsan-standalone = "0.1.0"
[features]
rtsan = ["rtsan-standalone/enable"]
```

All macros and functions are zero-cost when the feature is disabled, so you can leave them in your code.

Here's an example of using RTSan to catch a real-time violation. First, we mark our audio processing function as nonblocking:

```rust
use rtsan_standalone::nonblocking;

#[nonblocking]
fn process(data: &mut [f32]) {
    let _ = vec![0.0; 16]; // oops! allocating memory in a real-time context
}
```

The sanitizer immediately catches this violation and produces a detailed error trace:

```sh
==283082==ERROR: RealtimeSanitizer: unsafe-library-call
Intercepted call to real-time unsafe function calloc in real-time context!
    #0 0x55c0c3be8cf2 in calloc /tmp/.tmp6Qb4u2/llvm-project/compiler-rt/lib/rtsan/rtsan_interceptors_posix.cpp:470:34
    #1 0x55c0c3be4e69 in alloc::alloc::alloc_zeroed::hf760e6484fdf32c8 /rustc/f6e511eec7342f59a25f7c0534f1dbea00d01b14/library/alloc/src/alloc.rs:170:14
    ...
```

## Special Case: Mutex

An interesting discovery was the different mutex implementations between Linux and macOS. On macOS, mutex operations use pthread_mutex_* calls, with the first lock operation triggering an allocation. Setting `RTSAN_OPTIONS=halt_on_error=false` reveals the sequence: alloc, lock, then free on unlock.

Linux uses a futex-based implementation with a CAS loop for lock attempts. It first spins for a limited cycle count before falling back to a system call. RTSan only detects the Linux implementation when it resorts to the system call. While using such a mutex in real-time contexts might be acceptable if the system call is avoided, this ventures into complex lock-free programming territory that's beyond my current expertise. If you're knowledgeable about Linux mutex implementation, I'd love to hear your thoughts.

## Extended Features

Check the examples for additional features:

- The `blocking` macro marks functions as non-real-time-safe
- `scoped_disabler` temporarily disables sanitizer checks within nonblocking functions
- `no_sanitize_realtime` disables checks for entire functions
- RTSan behavior can be customized via [runtime flags](https://clang.llvm.org/docs/RealtimeSanitizer.html#run-time-flags)

## Get Involved

Thanks for reading! If you want to contribute or provide feedback:

- **Discord:** [RealtimeSanitizer (RTSan)](https://discord.com/invite/DZqjbmSZzZ)
- **Email:** [realtime.sanitizer@gmail.com](mailto:realtime.sanitizer@gmail.com)
- **GitHub Issues:** [Submit queries or suggestions](https://github.com/realtime-sanitizer/rtsan-standalone-rs)
