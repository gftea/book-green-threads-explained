# An example we can build upon

{% hint style="info" %}
In this example we will create our own stack and make our CPU return out of it's current execution context and over to the stack we just created. We will build on these concepts in the following chapters (we will not build upon the code though).
{% endhint %}

## Setting up our project

First let's start a new project in a folder named "green\_threads". Run:

`cargo init`

We need to use the Rust Nightly since we will use some features that are not stabilized yet:

`rustup override set nightly`

In our `main.rs` we start by importing the `asm!`macro:

{% code title="main.rs" %}
```rust
use core::arch::asm;
```
{% endcode %}

Let's set a small stack size here, only 48 bytes so we can print the stack and look at it before we switch contexts:

{% code title="main.rs" %}
```rust
const SSIZE: isize = 48;
```
{% endcode %}

{% hint style="warning" %}
There seems to be an issue in OSX using such a small stack. The minimum for this code to run is a stack size of 624 bytes . The code works on [Rust Playground](https://play.rust-lang.org) as written here if you want to follow this exact example (however you'll need to wait \~30 seconds for it to time out due to our loop in the end).
{% endhint %}

Then let's add a struct that represents our CPU state. We only focus on the register that stores the "stack pointer" for now since that is all we need:

{% code title="main.rs" %}
```rust
#[derive(Debug, Default)]
#[repr(C)]
struct ThreadContext {
    rsp: u64,
}
```
{% endcode %}

In later examples, we will use all the registers marked as "callee saved" in the specification document I linked to. These are the registers described in the x86-64 ABI that we'll need to save our context, but right now we only need one register to make the CPU jump over to our stack.

Note that this needs to be `#[repr(C)]` because we access the data the way we do in our assembly. Rust doesn't have a stable ABI, so there is no way for us to be sure that this will be represented in memory with `rsp` as the first 8 bytes. C has a stable ABI and that's exactly what this attribute tells the compiler to use. Granted, our struct only has one field right now, but we will add more later.

{% code title="main.rs" %}
```rust
fn hello() -> ! {
    println!("I LOVE WAKING UP ON A NEW STACK!");

    loop {}
}
```
{% endcode %}

For this very simple example, we will define a function that just prints out a message and then loops forever:

Next up is our inline assembly, where we switch over to our own stack.

```rust
unsafe fn gt_switch(new: *const ThreadContext) {
    asm!(
        "mov rsp, [{0} + 0x00]",
        "ret",
        in(reg) new,
    );
}
```

We use a trick here. We write the address of the function we want to run on our new stack. Then we pass the address of the first byte where we stored this address to the `rsp` register (the value we set to `new.rsp` will be _an address located on our own stack that stores the address of the `hello` function above_). Got it?

The `ret` keyword transfers program control to the return address located on top of the stack. Since we pushed our address to the `rsp` register, the CPU will think that is the return address of the function it's currently running, so when we pass the `ret` instruction it returns directly into our own stack.

The first thing the CPU does is read the address of our function and runs it.

## Quick introduction to Rusts inline assembly macro

If you haven't used inline assembly before this might look foreign, but we'll use an extended version of this later to switch contexts, so I'll explain what we're doing line by line:

`unsafe` is a keyword that indicates that Rust cannot enforce the safety guarantees in the function we write. Since we are manipulating the CPU directly, this is most definitely unsafe

```rust
gt_switch(new: *const ThreadContext)
```

Here we take a pointer to an instance of our `ThreadContext` from which we will only read one field.

```rust
asm!(
```

This is the `asm!`macro in the Rust standard library. It will check our syntax and provide an error message if it encounters something that doesn't look like valid Intel (by default) assembly syntax.

The first thing the macro takes as input is the assembly template:

```rust
"mov rsp, [{0} + 0x00]",
```

This is a simple instruction that moves the value stored at `0x00` offset (that means no offset at all in hex) from the memory location at `{0}` to the `rsp` register. Since the `rsp` register stores a pointer to the next value on the stack, we effectively push the address we provide it on top of the current stack, overwriting what's already there.

{% hint style="success" %}
Note that we don't need to write `[{0} + 0x00]` when we don't want an offset from the memory location. Writing `mov rsp, [{0}]`would be perfectly fine. However, I chose to introduce offset parameter here as we'll need it later on.
{% endhint %}

Note that the Intel syntax is a little "backwards". You might be tempted to think `mov a, b`means "move what's at `a` to `b`" but the Intel dialect usually dictates that the destination register is first and the source second.&#x20;

To make this confusing, this is the opposite of what's typically the case with the `AT&T`syntax, where reading it as "move `a`to`b`" is the correct thing to do. This is one of the fundamental differences between the two dialects, and it's useful to be aware of.

You will not see `{0}` used like this in normal assembly code. This is part of the assembly template and is a placeholder for the value passed as the first parameter to the macro. You'll notice that this closely matches how string templates are formatted in Rust using `println!`or the like. The parameters are numbered from 0, 1, 2…. We only have one input parameter here, which corresponds to `{0}`.&#x20;

You don't really have to index your parameters like this, writing `{}`in the correct order would suffice (as you would do using the `println!`macro). However, using an index improves readability and I would strongly recommend doing it that way.

The `[]` basically means: "get what's at this memory location", you can think of it as the same as dereferencing a pointer. To try to sum up what we do here with words: "move what's at the `+ 0x00` offset from the memory location that `{compiler_chosen_general_purpose_register}`points to, to the `rsp`register".

```rust
"ret",
```

The `ret` keyword instructs the CPU to pop a memory location off the top of the stack and then makes an unconditional jump to that location. In effect, we have hijacked our CPU and made it return to our stack.

{% code title="input" %}
```rust
in(reg) new,
```
{% endcode %}

The first non-assembly argument to the `asm!`macro is our `input` parameter. When we write `in(reg)` we let the compiler decide on a general purpose register to store the value of `new`. `out(reg)` means that the register is an output, so if we write `out(reg) new` we need `new` to be `mut` so we can write a value to it. You'll also find other versions like `inout` and `lateout`&#x20;

Inline assembly is quite complex, so we'll take this step by step and introduce the details on how these parameters are used gradually as we use them in our code.

#### Options

The last thing we need to introduce to get a minimal understanding of Rust's inline assembly for now is the `options` keyword. After the input and output parameters you'll often see something like `options(att_syntax)`which specifies that the assembly is written with the AT\&T syntax instead of the Intel syntax. Other options include `pure,` `nostack` and several others.

I'll refer you [to the documentation to read about them](https://doc.rust-lang.org/unstable-book/library-features/asm.html#options-1) since they're explained there.

## Running our example

```rust
fn main() {
    let mut ctx = ThreadContext::default();
    let mut stack = vec![0_u8; SSIZE as usize];

    unsafe {
        let stack_bottom = stack.as_mut_ptr().offset(SSIZE);
        let sb_aligned = (stack_bottom as usize & !15) as *mut u8;
        std::ptr::write(sb_aligned.offset(-16) as *mut u64, hello as u64);
        ctx.rsp = sb_aligned.offset(-16) as u64;
        gt_switch(&mut ctx);
    }
}
```

So this is actually designing our new stack. `hello` is a pointer already (a function pointer) so we can cast it directly as an `u64` since all pointers on 64 bits systems will be, well, 64 bit, and then we write this pointer to our new stack.

{% hint style="info" %}
We'll talk more about the stack in the next chapter but one thing we need to know already now is that the stack grows downwards. If our 48 byte stack starts at index _0_, and ends on index _47,_ index _32_ will be the first index of a 16 byte offset from the start/base of our stack.
{% endhint %}

Make note that we write the pointer to an offset of 16 bytes from the base of our stack (remember what I wrote about 16 byte alignment?).

{% hint style="info" %}
What does the line`let sb_aligned = (stack_bottom as usize &! 15) as *mut u8;`do?

When we ask for memory like we do when creating a `Vec<u8>`, there is no guarantee that the memory we get is 16-byte aligned when we get it. This line of code essentially rounds our memory address down to the nearest 16-byte aligned address. If it's already 16-byte aligned it does nothing.
{% endhint %}

We cast it as a pointer to an `u64` instead of a pointer to a `u8`. We want to write to position 32, 33, 34, 35, 36, 37, 38, 39 which is the 8 byte space we need to store our `u64`. If we don't do this cast we try to write an u64 only to position 32 which is not what we want.

We set the `rsp` (Stack Pointer) to _the memory address of index 32 in our stack_, we don't pass the value of the `u64` stored at that location but an address to the first byte.

When we `cargo run` this code we get:

```
Finished dev [unoptimized + debuginfo] target(s) in 0.58s
Running `target\debug\green_thread_start.exe`
I LOVE WAKING UP ON A NEW STACK!
```

OK, so what happened? We didn't call the function `hello` at any point, but it still was run. What happened is that we actually made the CPU jump over to our own stack and execute code there. We have taken the first step towards implementing a context switch.

In the next chapters we will talk about the stack a bit before we implement our green threads, it will be easier now that we have covered so much of the basics.
