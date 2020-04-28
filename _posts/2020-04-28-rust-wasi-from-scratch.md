---
layout: post
title: 'Stubbing out WASI manually in Rust'
---

# {{ page.title }}

Following a really cool blog post ["A beginner's guide to adding a new WASI syscall in Wasmtime"]
by [@RaduM](https://twitter.com/matei_radu) on adding a new WASI syscall to the WASI spec and then stubbing it
out in [Wasmtime](https://wasmtime.dev), I thought I'll contribute to his
effort in making Wasm and WASI simpler for the beginners, and delve a bit deeper into stubbing out WASI imports
manually when compiling from Rust using the bare target `wasm32-unknown-unknown`. I guess I'm
hopeful this would hint everyone on how [wasm-bindgen](https://github.com/rustwasm/wasm-bindgen) is doing
it, or what is actually happening under-the-hood when compiling directly to WASI in Rust (i.e., using the
target `wasm32-wasi`).

["A beginner's guide to adding a new WASI syscall in Wasmtime"]: https://radu-matei.com/blog/adding-wasi-syscall/

## Uhm, so what is it that you're actually trying to show here?

Excellent question! So here's the problem we're trying to solve here. Given a simple, bare Rust lib looking
as follows:

{% highlight rust %}
fn say_hello() {
  println!("Hello WASI!")
}
{% endhighlight %}

how can we configure and stub it out so that when compiled to `wasm32-unknown-unknown` it's actually run as
a standard executable WASI module? Incidentally, the solution to this problem is pretty straightforward once
you know what the WASI spec requires, and how to link to the runtime-provided syscall module. But, one thing at a time.

## The solution, step-by-step

### Step 0. Get the right tools!

You'll need:

* > [Wasmtime](https://github.com/bytecodealliance/wasmtime/releases/tag/v0.15.0)
* > [Rust toolchain](https://rustup.rs/)
* > `wasm32-unknown-unknown` target installed:

{% highlight console %}
$ rustup target add wasm32-unknown-unknown
{% endhighlight %}

After you've gathered all three, go ahead and create a Rust library template with:

{% highlight console %}
$ cargo new --lib hello-wasi
{% endhighlight %}

### Step 1. Configure Cargo.toml

What we actually want here is for the library to compile into a dynamic system library, or `cdylib`.
To do that, we need to tweak `Cargo.toml` to have the following `[lib]` section added:

{% highlight toml %}
# Cargo.toml
[package]
name = "hello-wasi"
version = "0.1.0"
authors = ["Jakub Konka <kubkon@jakubkonka.com>"]
edition = "2018"

[lib]
crate-type = ["cdylib"] # <-- this is the bit we need to add

[dependencies]
{% endhighlight %}

`cdylib` will allow us to build a dynamically linked Wasm module. With the right entrypoint specified, this will
allows to use it as a standard executable Wasm module that you would get by compiling a binary Rust crate
to `wasm32-wasi` target directly. You can find more on this [here](https://doc.rust-lang.org/reference/linkage.html).

### Step 2. Add the required entrypoint

At the time of writing, for a Wasm module to classify as a WASI module, it needs to export an entrypoint 
called `_start`. Cool, this is easy, let's add that in to our `lib.rs` source:

{% highlight rust %}
#[no_mangle]
pub unsafe extern "C" fn _start() {

}
{% endhighlight %}

Note the `#[no_mangle]` attribute which will basically stop Rust compiler from mangling the function's name,
and `extern "C"` since we need to generate ABI that our Wasm runtime (in this case, `Wasmtime`) can link to.

### Step 3. Print "Hello WASI!" to screen

OK, so this is one is the trickiest step so far. Since we're stubbing out the WASI imports/exports ourselves,
`println!` macro won't work as it has no idea where to redirect the output to. If we compiled to `wasm32-wasi`
that would have been taken care of for us, but in this case, we'll have to do this manually ourselves.

We'll need to import two WASI syscalls provided by the runtime when executing our module. These are:
* > `fd_write` -- allows writing to a WASI file descriptor (in our case, we'll redirect to stdout)
* > `proc_exit` -- terminates the process with the specified exit code

They're both provided as part of the `wasi_snapshot_preview1` module which we'll link to in our module.
Oh, and btw, you can explore the WASI spec, and in particular, `wasi_snapshot_preview1` module in more 
detail [here].

[here]: https://github.com/WebAssembly/WASI/blob/master/phases/snapshot/docs.md#-fd_writefd-fd-iovs-ciovec_array---errno-size

#### Step 3.1. Declaring the imports

We'll need the following set of imports declared and types/structs defined:

{% highlight rust %}
type Fd = u32;
type Size = usize;
type Errno = i32;
type Rval = u32;

#[repr(C)]
struct Ciovec {
    buf: *const u8,
    buf_len: Size,
}

#[link(wasm_import_module = "wasi_snapshot_preview1")]
extern "C" {
    fn fd_write(fd: Fd, iovs_ptr: *const Ciovec, iovs_len: Size, nwritten: *mut Size) -> Errno;
    fn proc_exit(rval: Rval);
}
{% endhighlight %}

A word of explanation what the heck is going on here. In order to link with the syscalls provided by
the runtime, we need to match the function name and its signature. For the latter to work out, we need
to be careful to properly specify the sizes (in bytes) of the input types, and so, under-the-hood,
the runtime is expecting a function `fd_write` to have the signature

{% highlight none %}
(i32, i32, i32, i32) -> i32
{% endhighlight %}

`i32` here means that every type is 4 bytes aligned.

#### Step 3.2. Using the imports to stub out `say_hello`

OK, with this out of the way, we can finally stub out `say_hello` function with a working replacement
of `println!`:

{% highlight rust %}
unsafe fn say_hello() -> Errno {
   let text = "Hello WASI!";
   let ciovec = Ciovec {
       buf: text.as_ptr(),
       buf_len: text.len(),
   };
   let ciovecs = [ciovec];
   let mut nwritten = 0;
   fd_write(1, ciovecs.as_ptr(), ciovecs.len(), &mut nwritten)
}
{% endhighlight %}

And our entrypoint:

{% highlight rust %}
#[no_mangle]
pub unsafe extern "C" fn _start() {
  let ret = say_hello();
  proc_exit(ret as Rval)
}
{% endhighlight %}

And that's it! We can compile our module and run using `Wasmtime`.

### Step 4. Compile for `wasm32-unknown-unknown`

{% highlight console %}
$ cargo build --target wasm32-unknown-unknown
{% endhighlight %}

If there were no errors, you should find your Wasm module in `target/wasm32-unknown-unknown/debug/hello_wasi.wasm`.

### Step 5. Run using `Wasmtime`

{% highlight console %}
$ wasmtime target/wasm32-unknown-unknown/debug/hello_wasi.wasm
Hello WASI!
{% endhighlight %}

To check that we're actually calling the syscalls from our module, we can enable the syscall trace like so:

{% highlight console %}
$ RUST_LOG=wasi_common=trace wasmtime target/wasm32-unknown-unknown/debug/hello_wasi.wasm
 DEBUG wasi_common::ctx > WasiCtx inserting entry PendingEntry::Thunk(0x7ffee24e04c8)
 DEBUG wasi_common::sys::unix::oshandle > Host fd 0 is a char device
 DEBUG wasi_common::sys::unix::oshandle > Host fd 0 is a char device
 DEBUG wasi_common::ctx                 > WasiCtx inserted at Fd(0)
 DEBUG wasi_common::ctx                 > WasiCtx inserting entry PendingEntry::Thunk(0x7ffee24e04c8)
 DEBUG wasi_common::sys::unix::oshandle > Host fd 1 is a char device
 DEBUG wasi_common::sys::unix::oshandle > Host fd 1 is a char device
 DEBUG wasi_common::ctx                 > WasiCtx inserted at Fd(1)
 DEBUG wasi_common::ctx                 > WasiCtx inserting entry PendingEntry::Thunk(0x7ffee24e04c8)
 DEBUG wasi_common::sys::unix::oshandle > Host fd 2 is a char device
 DEBUG wasi_common::sys::unix::oshandle > Host fd 2 is a char device
 DEBUG wasi_common::ctx                 > WasiCtx inserted at Fd(2)
 DEBUG wasi_common::old::snapshot_0::ctx > WasiCtx inserting (0, Some(PendingEntry::Thunk(0x7ffee24e5db0)))
 DEBUG wasi_common::old::snapshot_0::sys::unix::entry_impl > Host fd 0 is a char device
 DEBUG wasi_common::old::snapshot_0::ctx                   > WasiCtx inserting (1, Some(PendingEntry::Thunk(0x7ffee24e5dc0)))
 DEBUG wasi_common::old::snapshot_0::sys::unix::entry_impl > Host fd 1 is a char device
 DEBUG wasi_common::old::snapshot_0::ctx                   > WasiCtx inserting (2, Some(PendingEntry::Thunk(0x7ffee24e5dd0)))
 DEBUG wasi_common::old::snapshot_0::sys::unix::entry_impl > Host fd 2 is a char device
 TRACE wasi_common::wasi::wasi_snapshot_preview1           > fd_write(fd=Fd(1),iovs=*guest 0xfffd8/1)
Hello WASI! TRACE wasi_common::wasi::wasi_snapshot_preview1           >      | result=(nwritten=11)
 TRACE wasi_common::wasi::wasi_snapshot_preview1           >      | errno=No error occurred. System call completed successfully. (Errno::Success(0))
 TRACE wasi_common::wasi::wasi_snapshot_preview1           > proc_exit(rval=0)
{% endhighlight %}

## Final words

I hope this has been useful for you. If I have made a mistake anywhere (which I most likely have),
please feel free to reach out to me via any means available :-)
