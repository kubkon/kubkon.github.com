---
layout: post
title: Disable ASLR when debugging with LLDB on macOS
---

# {{ page.title }}

### TL;DR
*If you want to disable ASLR in LLDB, set the following option:*

```console
(lldb) settings set target.disable-aslr false
```

### The long version

This is one of these weird, short posts that the vast majority would not even think of doing since, in most
cases, it actually hinders the debugging process of your binary, but here goes. As you might be aware,
for a few weeks now, I'm on a mission to write an LLD replacement in Zig, [the ZigLd or zld](https://github.com/kubkon/zld)
with a first-class support for Mach-O linking and special focus on cross-compilation support (compile anywhere,
run on Apple Silicon, or the like). I've made some cool progress recently where I am almost able to link a basic
(which is not that basic if you take a closer look at) Zig hello-world-style binary. However, when executed
I'm greeted with a very informative (not!) segmentation fault message:

```console
> zig build-obj hello.zig
> zld hello.o compiler_rt.o -o hello
> ./hello
[1]    63559 segmentation fault  ./hello
```

You'd think, just stick it in a debugger to see where you've messed up address relocations, right? Well...

```console
> lldb ./hello
(lldb) r
Process 63694 launched: '/Users/kubkon/dev/zld/examples/hello' (arm64)
info: Hello, World!

Process 63694 exited with status = 0 (0x00000000)
```

Hmm, it works! Wait, what? Why? So it took me a while to realise that when the executable is run via LLDB,
LLDB by default turns off Address Space Layout Randomisation (ASLR)! Normally, this is a great thing since
your binary is loaded into the memory as if it wasn't a position-independent executable (PIE). In other words,
the address ranges for your local symbols you see when dissecting the disassembled machine code via `otool`
or `MachOView.app` is preserved. Great! Well, not in our case it turns out. In our case, there is some
addressing problem with our binary which only surfaces when the ASLR is on (which BTW is on by default on macOS,
and especially on Apple Silicon based Macs). In our case, we'd want to replicate the segfault. Well, fret not,
it turns out this can be done with the LLDB, and here's how. Let's redo our debugging session but this time
we'll tell LLDB not to disable the ASLR:

```console
> lldb ./hello
(lldb) settings set target.disable-aslr false
(lldb) r
Process 63723 launched: '/Users/kubkon/dev/zld/examples/hello' (arm64)
Process 63723 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=2, address=0x16b497ff0)
    frame #0: 0x0000000104170270 hello`std.debug.panicExtra + 44
hello`std.debug.panicExtra:
->  0x104170270 +44><: str    x0, [sp, #0x20]
    0x104170274 <+48>: add    x8, sp, #0x54             ; =0x54
    0x104170278 <+52>: str    x8, [sp, #0x18]
    0x10417027c +56><: str    x1, [sp, #0x10]
Target 0: (hello) stopped.
```

Yay! ASLR has not been disabled and we've successfully reproduced the segfault conditions, which means we
can hopefully work out what went wrong.
