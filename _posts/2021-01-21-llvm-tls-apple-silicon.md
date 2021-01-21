---
layout: post
title: "Thread-local storage, LLVM, and Apple Silicon: What can go wrong?"
---

# {{ page.title }}

The release of Apple Silicon in the form of the M1 chip has definitely stirred things up a lot! M1 boasts some
scary performance gains over Intel's even the most powerful i9 chips, while at the same time using much less energy. But,
while the hardware is something to look up to, the decision to switch to the ARM architecture and instruction set, every
day unravels more and more problems in existing software. In this short post, I'll describe the root cause for the
thread-local storage (TLS) miscompilation (or should I say, mislinking since this is a linker problem) by the LLVM
toolchain on macOS. For context, have a look at this outstanding issue in the Zig compiler:
[ziglang/zig#7527](https://github.com/ziglang/zig/issues/7527). I should point out here that Zig's stage1 (i.e.,
not-self-hosted) compiler uses LLVM for lowering the intermediate representation into the actual ISA. The self-hosted
compiler in Debug mode will by default use our own, in-house incremental linker, whereas in
Release mode the plan is to use LLVM to leverage years of really clever micro-optimisations that went into it.
(I promise I will write an update about the progress with the incremental Mach-O linker soon-ish, and I will make
sure to include all the juicy details that I learnt/discovered along the way. In the meantime, please join me at
my [FOSDEM21 session](https://fosdem.org/2021/schedule/event/zig_macho/) where I give an overview of the current
state-of-the-art.)

**DISCLAIMER:** Everything written here is the result of me reverse engineering different Mach-O binaries that I crafted
using different tools to try and pinpoint differences between programs that work and those that don't. For this reason,
I don't expect everything I describe here to always be technically accurate. Unfortunately, the docs on the modern
Mach-O are very scarce and so naturally, when writing the incremental linker in Zig, I've learnt most of the tricks
and built up my understanding of the entire file format by reverse engineering different examples.

**TL;DR** take everything you read here with a hefty pinch of salt!

## The scenario

Take this simple Zig source as an example:

{% highlight zig %}
threadlocal var x: usize = 0;

pub fn main() void {
    x = 1;
}
{% endhighlight %}

In this blog post, we will investigate how this source maps to the final Mach-O binary, and how a TLS variable
`x` is managed. I should point out here, that currently this example will lead to a segfault on Apple Silicon
for the reasons we'll discuss below.

## How is TLS laid out in the final Mach-O binary?

Before we dig deeper into the root cause of TLS miscompilation by the LLD, we should first of all get the basics
out of the way. If there is TLS within a Mach-O binary, the flags field within the Mach header must contain
the flag:

{% highlight zig %}
pub const MH_HAS_TLV_DESCRIPTORS: u32 = 0x800000;
{% endhighlight %}

Next, the `__DATA` segment (or more generally, the read-write segment) will contain two additional sections:
`__thread_vars` and `__thread_bss`. If you look closely, the former is the threaded equivalent of the `__data`
 section, while the latter of the `__bss` section.

### `__thread_vars` section

This section is identified by the flag:

{% highlight zig %}
pub const S_THREAD_LOCAL_VARIABLES: u32 = 0x13;
{% endhighlight %}

The section itself (or its actual contents) should be 8-byte aligned, and is usually padded with zeros. Here,
when the binary is loaded by the dynamic loader `dyld`, the `dyld_stub_binder` will write the address of the
`tlv_get_address` symbol, which the compiled machine code responsible for initialising the thread-local
variable `x`, will branch to, but more on this later.

For the `dyld_stub_binder` to know that it needs to populate the address of `tlv_get_address` function in the
space provided within the `__thread_vars` section, we use the Dynamic Loader Info (`LC_DYLD_INFO_ONLY` load command)
and in particular, the Binding Info section. Within the Binding Info, we guide the `dyld_stub_binder` to the right
segment and offset within that segment which should equal a cell within the `__thread_vars` section. I am not going
to dig into the actual opcodes used within the Binding Info section to drive the `dyld` in this blog post. Instead,
I'd like to point everyone to a very good resource by Jonathan Levin [here](http://www.newosxbook.com/articles/DYLD.html).

### `__thread_bss` section

This section is also 8-byte aligned and contains zerofill for thread-local storage, much like the `__bss` section
for the static global variables. The section is identified by the flag:

{% highlight zig %}
pub const S_THREAD_LOCAL_ZEROFILL: u32 = 0x12;
{% endhighlight %}

Also, I should point out that this section doesn't have data representation within the actual Mach-O file, i.e., its
file offset points to the beginning of the binary. However, it is required to point to an unoccupied space within
the virtual memory.

## How do we initialise a thread-local variable in the compiled machine code?

This is actually more straightforward than you may think. Within the symbol that makes use of the TLS, we call
`tlv_get_address` to initialise the variable. How do we do that? Remember that when the binary was loaded,
`dyld_stub_binder` was run, and fetched the address of `tlv_get_address` and saved it within the `__thread_vars`
section. Therefore, all that we need to do is to fetch that address and branch to the actual symbol via this address.
For the sake of example, assume that we are currently at an address `0x100003CE0`, and suppose that the `__thread_vars`
section is at a virtual address of `0x100049180`, and that the `dyld_stub_binder` will store the address of
`tlv_get_address` symbol in the first cell of the section, so at the section's start address `0x100049180`. Then,
all we need to do is first of all, locate the memory page where `__thread_vars` is residing at, to then narrow down
to the actual cell, to finally load the address stored in that cell in some register we will branch from. How do we
do this with ARM64 ISA?

{% highlight asm %}
adrp x0, 70      ; locate the page where `__thread_vars` resides
add  x0, x0, 384 ; locate the start of `__thread_vars` section i.e., `0x100049180`
ldr  x8, [x0, 0] ; dereference the contents of `0x100049180` cell and store it in `x8` register
blr  x8          ; branch with link to address within `x8` register
{% endhighlight %}

Just to recap, if the address of the first instruction above is `0x100003CE0`, then `adrp` will load the address
of `0x100003CE0 + 70 * 0x1000 := 0x100049000`. Note that the result is truncated to the nearest 4KB page. Next, we
add `384` (`0x180` in hex) to the result, so `0x100049000 + 0x180 := 0x100049180` which is, surprise, surprise, the
start address of the `__thread_vars` section and the address of the first cell of that section. Perfect! Then, we simply
derefence the value stored in `0x100049180`, which by this time, will be populated by the `dyld_stub_binder` and will
hold the actual address of the `tlv_get_address` symbol, and store it in `x8` register. Finally, we branch to it.

## What does the corresponding snippet look like when generated by the LLD?

Unfortunately, the output generated by the LLD uses `ldr` instead of `add` instruction leading to a segfault. Why?
First of, this is the output generated by the LLD:

{% highlight asm %}
adrp x0, 70      
ldr  x0, [x0, 384] ; Oh no! This is incorrect!
ldr  x8, [x0, 0] 
blr  x8          
{% endhighlight %}

In the above snippet, the use of `ldr` as the second instruction will first derefence the contents of memory at
an address `0x100049000` and *then* offset that value by `384`, whereas, what should happen is the exact opposite. We
should offset the address we want to load from by `384` as we did in the original snippet. This will lead to us
branching into whatever garbage was stored in `0x100049000` offset by `384` (or `0x180` in hex), and hence, will most
likely lead to a segfault or undefined behaviour.

## Why is this particularly important for Zig?

I have to admit, I'm using my M1 MBA to drive the development of the incremental linker, and for some time now, I've
been struggling by not having any stack traces in case I was hitting an assert or panicking somewhere in the codebase.
I didn't investigate the exact cause until very recently, and it turns out, miscompilation of TLS by LLD is to blame.
This is because, in Zig, every thread keeps track of its current panic stage. We achieve this via TLS:

{% highlight zig %}
// lib/std/debug.zig, line 242
threadlocal var panic_stage: usize = 0;
{% endhighlight %}

Therefore, any panic would have to traverse the corrupted codepath that we examined in the snippet above leading
to a segfault without printing any stack trace. Let me bring up an example. Consider this simple Zig snippet:

{% highlight zig %}
pub fn main() void {
    @panic("NO!");
}
{% endhighlight %}

Compiling and running the resulting binary, will unfortunately currently result in:

{% highlight console %}
❯ ./simple
[1]    31670 segmentation fault  ./simple
{% endhighlight %}

As an experiment, I decided to hack a manual fix for this by rewriting the offending instruction from `ldr` to `add`,
and the result is now as expected:

{% highlight console %}
thread 4981258 panic: NO!
/Users/kubkon/dev/zig/examples/st.zig:2:5: 0x10070feff in main (st)
    @panic("NO!");
    ^
/Users/kubkon/dev/zig/build/lib/zig/std/start.zig:335:22: 0x10071004b in std.start.main (st)
        root.main();
                 ^
???:?:?: 0x181b0cf33 in ??? (???)
Panicked during a panic. Aborting.
[1]    31664 abort      ./st
{% endhighlight %}

## What's next for Zig?

Clearly, a post-mortem fixup to the binary is not an option since this would require disassembling the entire program
searching for any mention of the TLS. An intermediate solution for the time being will be for us to not use TLS
on Apple Silicon until the problem is fixed in the LLD itself, or our in-house linker is able to perform well enough
as a drop-in replacement.
