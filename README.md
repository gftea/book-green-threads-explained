---
description: >-
  This book aims to explain green threads by using a small example where we
  implement a simple but working program where we use our own green threads to
  execute code.
---

# Introduction

{% hint style="info" %}
All the code we go through here is [located in a github repository](https://github.com/cfsamson/example-greenthreads). There are two branches, the `main` branch that only contains the code and the `commented` branch that contains the code with comments explaining what we do.
{% endhint %}

Green threads, userland threads, goroutines or fibers, they have many names but for simplicity's sake I'll refer to them all as green threads from now on.

In this article I want to explore how they work by implementing a very simple example where we create our own green threads in 200 lines of Rust code. We'll be explaining everything along the way so our main focus here is to understand them and learn how they work by using simple, but working example.

{% hint style="info" %}
We will not use any external libraries or helpers and will do everything from scratch so we make sure we really understand what's going on.
{% endhint %}

## Who is this article for?

We are peeking down the rabbit hole in this article so if that sounds scary, this article probably isn't for you. Just go back and live happily ever after.

If you are the curious kind and want to understand how things work, then read on. Maybe you've heard of Go and its goroutines, or the equivalent in Ruby or Julia and you know how to use them but want to know how they work - well then read on.

In addition, this should be interesting if:

* You're new to Rust and want to learn more about its features.
* You want to learn more about how operating systems work
* You want to learn the basics of inline assembly in Rust.
* Or, you're just curious.&#x20;

Well, join me as we try to figure out everything we need to understand them.

You don't have to be a Rust programmer to understand this article, but it is highly recommended to read some of the basic syntax first. If you want to follow a long or clone the repo and play around with the code, you should probably get Rust and learn the basics.

{% hint style="info" %}
[You will find everything you need to set up Rust here.](https://www.rust-lang.org/tools/install)
{% endhint %}

## Following along

All the code I provide here is in a single file and has no dependencies, which means that you can easily start your own project and follow along if you want to (I suggest you do). You can even run most of the code in the [Rust playground](https://play.rust-lang.org). Just remember to use the `nightly` version of the compiler.

## Disclaimer <a href="#docs-internal-guid-12e6c217-7fff-3de7-4bee-4532b47ef574" id="docs-internal-guid-12e6c217-7fff-3de7-4bee-4532b47ef574"></a>

I'm not trying to make a perfect implementation here. I'm cutting corners to get down to the essence and fit it into what was originally intended to be an article but expanded into a small book. This is not the best way of displaying Rusts greatest strengths, its safety guarantees, but it does show an interesting use of Rust and the code is mostly pretty clean and easy to follow.

However, if you spot places where I can make the code safer without making it significantly more complex, I welcome you to create an issue in [the repo](https://github.com/cfsamson/example-greenthreads) or even better, a pull request.

## Credits

[Quentin Carbonneaux](https://github.com/mpu) wrote a [nice article](https://c9x.me/articles/gthreads/intro.html) back in 2013 which I used as inspiration for the main code example. Thanks to [nickelpro](https://github.com/nickelpro) for help and feedback on Windows support.

## Edits

2022-01-27: Large rewrite of all the inline assembly to Rust's new syntax. Updated the introduction to inline assembly and the explanations throughout the book to cover the new `asm!`macro instead of the old LLVM syntax. Rewrote assembly from the AT\&T dialect which was the default on the old syntax to the Intel dialect which is the default on the new syntax. Fixed issues with `#[naked]` functions. All examples work as they should on the 1.60.0 nightly compiler.

2020-08-04: Thanks to [ziyi-yan](https://github.com/ziyi-yan), which identified an issue with the `guard` function not being 16-byte aligned, the example is now correct and can be used as a basis for more advanced experimentation. See the relevant [issue](https://github.com/cfsamson/example-greenthreads/issues/19) in the example repo for more information.

2020-05-20: Changed to the `llvm_asm!` macro since we use the syntax used by llvm in this book. The new Rust syntax for inline assembly has now been merged into the nightly compiler and uses the `asm!` macro, which would have caused problems for the examples in this book.

2019-06-18: New chapter implementing a proper Windows support

2019-06-21: Rather substantial change and cleanup. An issue was reported that Valgrind reported some troubles with the code and crashed. This is now fixed and there are currently no unsolved issues. In addition, the code now runs on both `debug`and `release`builds without any issues on all platforms. Thanks to everyone for reporting issues they found.

2019-06-26: The Supporting Windows appendix treated the `XMM`fields as 64 bits, but they are 128 bits which was an oversight on my part. Correcting this added some interesting material to that chapter but unfortunately also some complexity. However, it's now corrected and explained.

2019-22-12: Added one line of code to make sure the memory we get from the allocator is 16 byte aligned. Refactored to use the "high" memory address as basis for offsets when writing to the stack since this made alignment easier. Thanks to [@Veetaha](https://github.com/Veetaha) for addressing this issue.
