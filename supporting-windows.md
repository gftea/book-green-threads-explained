# Appendix: Supporting Windows

Our example works for both OSX, Linux and Windows, but as I have pointed out, our Windows implementation is not correct even though it's working. Since I've been quite commited to make this work on all three platforms, I'll go through what we need to do in this chapter.

You might wonder why I didn't include this in the original code, and the reason for that is that this is really not at all that relevant for explaining the main concepts I wanted to explore.

Here I'm trying to go a bit further here to explore how we should set up the stack for Windows correct way and do a proper context switch. Even though we might not get all the way to a perfect implementation, there is plenty of information and references for you to explore further and I'll list some of them here:

* [Microsofts x64 software conventions](https://docs.microsoft.com/en-us/cpp/build/x64-software-conventions?view=vs-2019)
* [Win64/AMD64 API](https://wiki.lazarus.freepascal.org/Win64/AMD64\_API) - nice summary of differences between psABi and Win64
* [Handmade Coroutines for Windows](https://probablydance.com/2013/02/20/handmade-coroutines-for-windows/) - a very good read about a coroutine implementation
* [Boost Context assembly](https://github.com/boostorg/context/blob/develop/src/asm/ontop\_x86\_64\_ms\_pe\_gas.asm) - it's in C++ but it's a good reference for further study

## What's special with Windows

The reason I don't consider this important enough to implement in the main example is that that windows has more `callee saved` registers, or `non-volatile`registers as they call it in addition to one rather poorly documented quirk that we need to account for, so what we really do is just to save more data when we do the context switch and that needs more conditional compilation.

{% hint style="info" %}
Conditionally compiling this to support windows correctly bloats our code with almost 50 % without adding much to what we need for a basic understanding.

Now that doesn't mean this isn't interesting, on the contrary, but we'll also experience first hand some of the difficulties of supporting multiple platforms when doing everything from scratch.
{% endhint %}

## Additional callee saved (non-volatile) registers

The first thing I mentioned is that windows wants to save more data during context switches, in particular the XMM6-XMM15 registers. It's actually [mentioned specifically in the reference](https://docs.microsoft.com/en-us/cpp/build/x64-software-conventions?view=vs-2019#register-usage) so this is just adding more fields to our `ThreadContext` struct. This is very easy now that we've done it once before.

In addition to the `XMM`registers the `rdi`and `rsi`registers are nonvolatile on Windows which means they're callee saved (on linux these registers are use for the first and second function arguments), so we need to add these too.

However, there is one caveat: the `XMM`registers are 128 bits, and not 64. Rust has a `u128`type but we'll use `[u64;2]`instead to avoid some alignment issues that we _might_ get otherwise. Don't worry, I'll explain this further down.

Our ThreadContext now looks like this:

```rust
#[cfg(target_os="windows")]
#[derive(Debug, Default)]
#[repr(C)] 
struct ThreadContext {
    rsp: u64,
    r15: u64,
    r14: u64,
    r13: u64,
    r12: u64,
    rbx: u64,
    rbp: u64,
    rdi: u64,
    rsi: u64,
    xmm6: [u64; 2],
    xmm7: [u64; 2],
    xmm8: [u64; 2],
    xmm9: [u64; 2],
    xmm10: [u64; 2],
    xmm11: [u64; 2],
    xmm12: [u64; 2],
    xmm13: [u64; 2],
    xmm14: [u64; 2],
    xmm15: [u64; 2],
}
```

## The Thread Information Block

The second part is poorly documented. I've actually struggled to verify exactly how skipping this will cause a failure on modern Windows but there's [enough references to it](https://probablydance.com/2013/02/20/handmade-coroutines-for-windows/) around from [trustworthy sources](https://github.com/boostorg/context/blob/develop/src/asm/ontop\_x86\_64\_ms\_pe\_gas.asm#L116-L129) that I'm in no doubt we need to go through this.

You see, Windows wants to store some information about the currently running thread in what it calls the `Thread Information Block`, referred to as `NT_TIB`. Specifically it wants access to information about the `Stack Base`and the `Stack Limit`in the `%gs`register.

{% hint style="info" %}
What is the GS register you might ask?

The answer I found was a bit perplexing. Apparently these segment registers, GS on x64, and FS on x86 was intended by Intel to [allow programs to access many different segments of memory](https://stackoverflow.com/questions/10810203/what-is-the-fs-gs-register-intended-for) that were meant to be part of a persistent virtual store. Modern operating systems doesn't use these registers this way as we can only access our own process memory (which appear as a "flat" memory to us as programmers). Back when it wasn't clear that this would be the prevailing model, these registers would allow for different implementations by different operating systems. See the [Wikipedia article on the Multics operating system](https://en.wikipedia.org/wiki/Multics) if you're curious.
{% endhint %}

That means that these segment registers are freely used by operating systems for what they deem appropriate. Windows stores information about the currently running thread in the GS register, and Linux uses these registers for thread local storage.

When we switch threads, we should provide the information it expects about our [Stack Base and our Stack Limit](https://en.wikipedia.org/wiki/Win32\_Thread\_Information\_Block).

Our `ThreadContext` now looks like this:

{% tabs %}
{% tab title="ThreadContext" %}
```rust
#[cfg(target_os="windows")]
#[derive(Debug, Default)]
#[repr(C)] 
struct ThreadContext {
    rsp: u64,
    r15: u64,
    r14: u64,
    r13: u64,
    r12: u64,
    rbx: u64,
    rbp: u64,
    rdi: u64,
    rsi: u64,
    xmm6: [u64; 2],
    xmm7: [u64; 2],
    xmm8: [u64; 2],
    xmm9: [u64; 2],
    xmm10: [u64; 2],
    xmm11: [u64; 2],
    xmm12: [u64; 2],
    xmm13: [u64; 2],
    xmm14: [u64; 2],
    xmm15: [u64; 2],
    stack_start: u64,
    stack_end: u64,
}
```
{% endtab %}
{% endtabs %}

{% hint style="info" %}
Notice we use the `#[cfg(target_os="windows")]` attribute here on all the Windows specific functions and structs, which mean we need to give our "original" definitions an attribute that makes sure it compiles them for all other targets than Windows: `[cfg(not(target_os="windows"))].`
{% endhint %}

I named the fields `stack_start` and `stack_end` since I find that easier to mentally parse since we know the stack starts on the top and grows downwards to the bottom.

Now to implement this we need to make a change to our `spawn()` function to actually provide this information:

## The Windows stack

![https://docs.microsoft.com/en-us/cpp/build/stack-usage?view=vs-2019#stack-allocation](<.gitbook/assets/image (1).png>)

You see, since Rust sets up our stack frames, we only need to care about where to put our `%rsp`and the return address and this looks pretty much the same as in the psABI. The differences between Win64 and psABI are elsewhere and Rust takes care of all these differences for us.

Now to implement this we need to make a change to our `spawn()`function to actually provide this information and set up our stack.

{% tabs %}
{% tab title="spawn" %}
```rust
 #[cfg(target_os = "windows")]
pub fn spawn(&mut self, f: fn()) {
    let available = self
        .threads
        .iter_mut()
        .find(|t| t.state == State::Available)
        .expect("no available thread.");

    let size = available.stack.len();

    // see: https://docs.microsoft.com/en-us/cpp/build/stack-usage?view=vs-2019#stack-allocation
    unsafe {
        let s_ptr = available.stack.as_mut_ptr().offset(size as isize);
        let s_ptr = (s_ptr as usize & !15) as *mut u8;
        std::ptr::write(s_ptr.offset((size - 16) as isize) as *mut u64, guard as u64);
        std::ptr::write(s_ptr.offset((size - 24) as isize) as *mut u64, skip as u64);
        std::ptr::write(s_ptr.offset((size - 32) as isize) as *mut u64, f as u64);
        available.ctx.rsp = s_ptr.offset((size - 32) as isize) as u64;
        available.ctx.stack_start = s_ptr as u64;

        available.ctx.stack_end = s_ptr as *const u64 as u64;
    }


    available.state = State::Ready;
}
```
{% endtab %}
{% endtabs %}

As you see we provide a pointer to the start of our stack and a pointer to the end of our stack.

### Possible alignment issues

Well, this is supposed to be hard, remember? Windows does not disappoint us making things too easy. You see, when we move data from a 128 bit register we need to use some special assembly instructions. There are several of them that _mostly_ do the same:

* `movdqa` [move double quad word aligned](https://www.felixcloutier.com/x86/movdqa:vmovdqa32:vmovdqa64)
* `movdqu`[move double quad word unaligned](https://www.felixcloutier.com/x86/movdqu:vmovdqu8:vmovdqu16:vmovdqu32:vmovdqu64)
* `movaps`[move aligned packed single-precision floating point value](https://www.felixcloutier.com/x86/movaps)
* `movups`[move unaligned packed single-precision floating point value](https://www.felixcloutier.com/x86/movups)

As you see most method has an `aligned`and and `unaligned`variant. The difference here is that `*ps`type instructions are targeting floating point values and the `*dq/*dq` type instructions are targeting integer values. Now both will work, but if you clicked on the Microsoft reference you probably noticed that the `XMM` are used for floating point values so the `*ps`type instructions are the right ones for us to use.

The `aligned`versions have historically been slightly faster under most circumstances and would be preferred in a context switch, but the latest information I've read about this is **that they've been practically the same for the last 6 generations of CPUs regarding performance**.

{% hint style="info" %}
If you want to read more about the cost for different instructions on newer and older processors, have a look at [Agner Fog's instruction tables](https://www.agner.org/optimize/instruction\_tables.pdf).
{% endhint %}

However, since the aligned instructions are used in all the reference implementations I've encountered, we'll use them as well although they expose us for some extra complexity, we are still learning stuff aren't we?

By aligned, we mean that the memory they read/write from/to is 16 byte aligned.

Now, the way I solve this is to push the fields that requires alignment to the start of our struct, and add a new attribute `#[repr(align(16))]`.

{% hint style="info" %}
The `repr(align(n))`attribute ensures that our struct starts at a 16 byte aligned memory address, so when we write to `XMM6`in the start of our assembly it's already 16 byte aligned, and since 128 is divisible with 16 so are the rest of the fields.

But, and this can be important, since we now have two different field sizes our compiler might choose to "pad" our fields, now that doesn't happen right now but pushing our larger fields to the start will minimize the risk of that happening at a later time.

We also avoid manually adding a padding member to our struct since we have 7`u64`fields before our`XMM`fields preventing them from aligning to 16 (remember, the`repr(C)`attribute guarantees that the compiler will not reorder our fields).
{% endhint %}

Our `Threadcontext`ends up like this after our changes:

{% tabs %}
{% tab title="ThreadContext" %}
```rust
#[cfg(target_os = "windows")]
#[derive(Debug, Default)]
#[repr(C)]
#[repr(align(16))]
struct ThreadContext {
    xmm6: [u64; 2],
    xmm7: [u64; 2],
    xmm8: [u64; 2],
    xmm9: [u64; 2],
    xmm10: [u64; 2],
    xmm11: [u64; 2],
    xmm12: [u64; 2],
    xmm13: [u64; 2],
    xmm14: [u64; 2],
    xmm15: [u64; 2],
    rsp: u64,
    r15: u64,
    r14: u64,
    r13: u64,
    r12: u64,
    rbx: u64,
    rbp: u64,
    rdi: u64,
    rsi: u64,
    stack_start: u64,
    stack_end: u64,
}
```
{% endtab %}
{% endtabs %}

Last we need to change our `switch()`function and update our assembly. After all this explanation this is pretty easy:

```rust
#[cfg(target_os = "windows")]
#[naked]
#[no_mangle]
unsafe fn switch() {
    asm!(
        "movaps      [rcx + 0x00], xmm6",
        "movaps      [rcx + 0x10], xmm7",
        "movaps      [rcx + 0x20], xmm8",
        "movaps      [rcx + 0x30], xmm9",
        "movaps      [rcx + 0x40], xmm10",
        "movaps      [rcx + 0x50], xmm11",
        "movaps      [rcx + 0x60], xmm12",
        "movaps      [rcx + 0x70], xmm13",
        "movaps      [rcx + 0x80], xmm14",
        "movaps      [rcx + 0x90], xmm15",
        "mov         [rcx + 0xa0], rsp",
        "mov         [rcx + 0xa8], r15",
        "mov         [rcx + 0xb0], r14",
        "mov         [rcx + 0xb8], r13",
        "mov         [rcx + 0xc0], r12",
        "mov         [rcx + 0xc8], rbx",
        "mov         [rcx + 0xd0], rbp",
        "mov         [rcx + 0xd8], rdi",
        "mov         [rcx + 0xe0], rsi",
        "mov         rax, gs:0x08",
        "mov         [rcx + 0xe8], rax",
        "mov         rax, gs:0x10",
        "mov         [rcx + 0xf0], rax",
        "movaps      xmm6, [rdx + 0x00]",
        "movaps      xmm7, [rdx + 0x10]",
        "movaps      xmm8, [rdx + 0x20]",
        "movaps      xmm9, [rdx + 0x30]",
        "movaps      xmm10, [rdx + 0x40]",
        "movaps      xmm11, [rdx + 0x50]",
        "movaps      xmm12, [rdx + 0x60]",
        "movaps      xmm13, [rdx + 0x70]",
        "movaps      xmm14, [rdx + 0x80]",
        "movaps      xmm15, [rdx + 0x90]",
        "mov         rsp, [rdx + 0xa0]",
        "mov         r15, [rdx + 0xa8]",
        "mov         r14, [rdx + 0xb0]",
        "mov         r13, [rdx + 0xb8]",
        "mov         r12, [rdx + 0xc0]",
        "mov         rbx, [rdx + 0xc8]",
        "mov         rbp, [rdx + 0xd0]",
        "mov         rdi, [rdx + 0xd8]",
        "mov         rsi, [rdx + 0xe0]",
        "mov         rax, [rdx + 0xe8]",
        "mov         gs:0x08, rax",
        "mov         rax, [rdx + 0xf0]",
        "mov         gs:0x10, rax",
        "ret", options(noreturn)
    );
}
```

As you see, our code gets just a little bit longer. It's not difficult once you've figured out what to store where, but it does add a lot of code.

{% hint style="warning" %}
Our inline assembly won't let us `mov` from one memory offset to another memory offset so we need to go via a register. I chose the`rax` register (the default register for the return value) but could have chosen any general purpose register for this.
{% endhint %}

## Conclusion

So this is all we needed to do. As you see we don't really do anything new here, the difficult part is figuring out how Windows works and what it expects, but now that we have done our job properly we should have a pretty complete context switch for all three platforms.

## Final code

I've collected all the code we need to compile differently for Windows at the bottom so you don't have to read the first 200 lines of code since that is unchanged from what we in the previous chapters. I hope you understand why I chose to dedicate a separate chapter for this.

{% hint style="info" %}
You'll also find this code in the [Windows branch in the repository](https://github.com/cfsamson/example-greenthreads/tree/windows).
{% endhint %}

```rust
use std::arch::asm;

const DEFAULT_STACK_SIZE: usize = 1024 * 1024 * 2;
const MAX_THREADS: usize = 4;
static mut RUNTIME: usize = 0;

pub struct Runtime {
    threads: Vec<Thread>,
    current: usize,
}

#[derive(PartialEq, Eq, Debug)]
enum State {
    Available,
    Running,
    Ready,
}

struct Thread {
    id: usize,
    stack: Vec<u8>,
    ctx: ThreadContext,
    state: State,
}

#[cfg(not(target_os = "windows"))]
#[derive(Debug, Default)]
#[repr(C)]
struct ThreadContext {
    rsp: u64,
    r15: u64,
    r14: u64,
    r13: u64,
    r12: u64,
    rbx: u64,
    rbp: u64,
}

impl Thread {
    fn new(id: usize) -> Self {
        Thread {
            id,
            stack: vec![0_u8; DEFAULT_STACK_SIZE],
            ctx: ThreadContext::default(),
            state: State::Available,
        }
    }
}

impl Runtime {
    pub fn new() -> Self {
        let base_thread = Thread {
            id: 0,
            stack: vec![0_u8; DEFAULT_STACK_SIZE],
            ctx: ThreadContext::default(),
            state: State::Running,
        };

        let mut threads = vec![base_thread];
        let mut available_threads: Vec<Thread> = (1..MAX_THREADS).map(|i| Thread::new(i)).collect();
        threads.append(&mut available_threads);

        Runtime {
            threads,
            current: 0,
        }
    }

    pub fn init(&self) {
        unsafe {
            let r_ptr: *const Runtime = self;
            RUNTIME = r_ptr as usize;
        }
    }

    pub fn run(&mut self) -> ! {
        while self.t_yield() {}
        std::process::exit(0);
    }

    fn t_return(&mut self) {
        if self.current != 0 {
            self.threads[self.current].state = State::Available;
            self.t_yield();
        }
    }

    #[inline(never)]
    fn t_yield(&mut self) -> bool {
        let mut pos = self.current;
        while self.threads[pos].state != State::Ready {
            pos += 1;
            if pos == self.threads.len() {
                pos = 0;
            }
            if pos == self.current {
                return false;
            }
        }

        if self.threads[self.current].state != State::Available {
            self.threads[self.current].state = State::Ready;
        }

        self.threads[pos].state = State::Running;
        let old_pos = self.current;
        self.current = pos;

        unsafe {
            let old: *mut ThreadContext = &mut self.threads[old_pos].ctx;
            let new: *const ThreadContext = &self.threads[pos].ctx;

            if cfg!(not(target_os = "windows")) {
                asm!("call switch", in("rdi") old, in("rsi") new, clobber_abi("C"));
            } else {
                asm!("call switch", in("rcx") old, in("rdx") new, clobber_abi("system"));
            }
        }

        // preventing compiler optimizing our code away on windows. Will never be reached anyway.
        self.threads.len() > 0
    }

    #[cfg(not(target_os = "windows"))]
    pub fn spawn(&mut self, f: fn()) {
        let available = self
            .threads
            .iter_mut()
            .find(|t| t.state == State::Available)
            .expect("no available thread.");

        let size = available.stack.len();
        unsafe {
            let s_ptr = available.stack.as_mut_ptr().offset(size as isize);
            let s_ptr = (s_ptr as usize & !15) as *mut u8;
            std::ptr::write(s_ptr.offset(-16) as *mut u64, guard as u64);
            std::ptr::write(s_ptr.offset(-24) as *mut u64, skip as u64);
            std::ptr::write(s_ptr.offset(-32) as *mut u64, f as u64);
            available.ctx.rsp = s_ptr.offset(-32) as u64;
        }
        available.state = State::Ready;
    }
}

#[naked]
unsafe fn skip() {
    asm!("ret", options(noreturn))
}


fn guard() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        (*rt_ptr).t_return();
    };
}

pub fn yield_thread() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        (*rt_ptr).t_yield();
    };
}

#[cfg(not(target_os = "windows"))]
#[naked]
#[no_mangle]
unsafe fn switch() {
    asm!(
        "mov [rdi + 0x00], rsp",
        "mov [rdi + 0x08], r15",
        "mov [rdi + 0x10], r14",
        "mov [rdi + 0x18], r13",
        "mov [rdi + 0x20], r12",
        "mov [rdi + 0x28], rbx",
        "mov [rdi + 0x30], rbp",
        "mov rsp, [rsi + 0x00]",
        "mov r15, [rsi + 0x08]",
        "mov r14, [rsi + 0x10]",
        "mov r13, [rsi + 0x18]",
        "mov r12, [rsi + 0x20]",
        "mov rbx, [rsi + 0x28]",
        "mov rbp, [rsi + 0x30]",
        "ret", options(noreturn)
    );
}

pub fn run() {
    let mut runtime = Runtime::new();
    runtime.init();
    runtime.spawn(|| {
        println!("THREAD 1 STARTING");
        let id = 1;
        for i in 0..10 {
            println!("thread: {} counter: {}", id, i);
            yield_thread();
        }
        println!("THREAD 1 FINISHED");
    });
    runtime.spawn(|| {
        println!("THREAD 2 STARTING");
        let id = 2;
        for i in 0..15 {
            println!("thread: {} counter: {}", id, i);
            yield_thread();
        }
        println!("THREAD 2 FINISHED");
    });
    runtime.run();
}

// ===== WINDOWS SUPPORT =====
#[cfg(target_os = "windows")]
#[derive(Debug, Default)]
#[repr(C)]
struct ThreadContext {
    xmm6: [u64; 2],
    xmm7: [u64; 2],
    xmm8: [u64; 2],
    xmm9: [u64; 2],
    xmm10: [u64; 2],
    xmm11: [u64; 2],
    xmm12: [u64; 2],
    xmm13: [u64; 2],
    xmm14: [u64; 2],
    xmm15: [u64; 2],
    rsp: u64,
    r15: u64,
    r14: u64,
    r13: u64,
    r12: u64,
    rbx: u64,
    rbp: u64,
    rdi: u64,
    rsi: u64,
    stack_start: u64,
    stack_end: u64,
}

impl Runtime {
    #[cfg(target_os = "windows")]
    pub fn spawn(&mut self, f: fn()) {
        let available = self
            .threads
            .iter_mut()
            .find(|t| t.state == State::Available)
            .expect("no available thread.");

        let size = available.stack.len();

        // see: https://docs.microsoft.com/en-us/cpp/build/stack-usage?view=vs-2019#stack-allocation
        unsafe {
            let s_ptr = available.stack.as_mut_ptr().offset(size as isize);
            let s_ptr = (s_ptr as usize & !15) as *mut u8;
            std::ptr::write(s_ptr.offset((size - 16) as isize) as *mut u64, guard as u64);
            std::ptr::write(s_ptr.offset((size - 24) as isize) as *mut u64, skip as u64);
            std::ptr::write(s_ptr.offset((size - 32) as isize) as *mut u64, f as u64);
            available.ctx.rsp = s_ptr.offset((size - 32) as isize) as u64;
            available.ctx.stack_start = s_ptr as u64;

            available.ctx.stack_end = s_ptr as *const u64 as u64;
        }


        available.state = State::Ready;
    }
}

// reference: https://probablydance.com/2013/02/20/handmade-coroutines-for-windows/
// Contents of TIB on Windows: https://en.wikipedia.org/wiki/Win32_Thread_Information_Block
#[cfg(target_os = "windows")]
#[naked]
#[no_mangle]
unsafe fn switch() {
    asm!(
        "movaps      [rcx + 0x00], xmm6",
        "movaps      [rcx + 0x10], xmm7",
        "movaps      [rcx + 0x20], xmm8",
        "movaps      [rcx + 0x30], xmm9",
        "movaps      [rcx + 0x40], xmm10",
        "movaps      [rcx + 0x50], xmm11",
        "movaps      [rcx + 0x60], xmm12",
        "movaps      [rcx + 0x70], xmm13",
        "movaps      [rcx + 0x80], xmm14",
        "movaps      [rcx + 0x90], xmm15",
        "mov         [rcx + 0xa0], rsp",
        "mov         [rcx + 0xa8], r15",
        "mov         [rcx + 0xb0], r14",
        "mov         [rcx + 0xb8], r13",
        "mov         [rcx + 0xc0], r12",
        "mov         [rcx + 0xc8], rbx",
        "mov         [rcx + 0xd0], rbp",
        "mov         [rcx + 0xd8], rdi",
        "mov         [rcx + 0xe0], rsi",
        "mov         rax, gs:0x08",
        "mov         [rcx + 0xe8], rax",
        "mov         rax, gs:0x10",
        "mov         [rcx + 0xf0], rax",
        "movaps      xmm6, [rdx + 0x00]",
        "movaps      xmm7, [rdx + 0x10]",
        "movaps      xmm8, [rdx + 0x20]",
        "movaps      xmm9, [rdx + 0x30]",
        "movaps      xmm10, [rdx + 0x40]",
        "movaps      xmm11, [rdx + 0x50]",
        "movaps      xmm12, [rdx + 0x60]",
        "movaps      xmm13, [rdx + 0x70]",
        "movaps      xmm14, [rdx + 0x80]",
        "movaps      xmm15, [rdx + 0x90]",
        "mov         rsp, [rdx + 0xa0]",
        "mov         r15, [rdx + 0xa8]",
        "mov         r14, [rdx + 0xb0]",
        "mov         r13, [rdx + 0xb8]",
        "mov         r12, [rdx + 0xc0]",
        "mov         rbx, [rdx + 0xc8]",
        "mov         rbp, [rdx + 0xd0]",
        "mov         rdi, [rdx + 0xd8]",
        "mov         rsi, [rdx + 0xe0]",
        "mov         rax, [rdx + 0xe8]",
        "mov         gs:0x08, rax",
        "mov         rax, [rdx + 0xf0]",
        "mov         gs:0x10, rax",
        "ret", options(noreturn)
    );
}

```
