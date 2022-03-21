---
layout: post
title: 'Hot-code reloading on macOS/arm64 with Zig - part 2: entitlements'
---

# {{ page.title }}

Last week I wrote about a proof-of-concept (poc) hot-code reloading on macOS/arm64 with Zig. By the way, if you didn't get
a chance to read it yet, you can do so by [following this link here]({% post_url 2022-03-16-hcs-zig %}). To summarise,
the poc was a success, and we have a way to perform hot-code reloading in Zig by directly writing to the running
process' memory. There was one huge problem though: we needed `sudo`. If you recall, the Mach mechanisms we employed
to get it to work are also used by debuggers on macOS including `lldb`. If you launch `lldb` right this instant,
you will notice you are *not* required to elevate your privileges with `sudo`. Huh. So how is that? Is it because
we are using Apple's `lldb` which would somehow be autotrusted by the OS? But wait, if I build stock `lldb` I can
still debug everything without `sudo` [^1]. The short answer to this is: _entitlements_!

### Entitlements

According to Apple's official docs, _an entitlement_ is [^2]

> a right or privilege that grants an executable particular capabilities. For example, an app needs the HomeKit
> Entitlement---along with explicit user consent---to access a user's home automation network. An app stores its
> entitlements as key-value pairs embedded in the code signature of its binary executable.

If you look closely in `lldb`'s sources [^1], you will notice a `*.plist` file in `resources/` subdir. If you `cat`
its contents, you will see something like this

```console
$ cat resources/debugserver-macosx-entitlements.plist
```

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.cs.debugger</key>
    <true/>
</dict>
</plist>

```

This file asks for `com.apple.security.cs.debugger` entitlement, and if we consult Apple's docs again [^3], we will
see that this is a _debugging tool entitlement_ which is

> a boolean value that indicates whether the app is a debugger and may attach to other processes or get task ports.

Woah---_may attach to other processes or get task ports_---this is exactly what we need for hot-code reloading! Actually,
this is somewhat of an overkill as we don't really want to attach to _any_ running process. We want to attach to a
child process that we, the compiler, spawned. Alas, I couldn't find an entitlement that would be less powerful, so this
will have to do for now.

So, we have found the right entitlement, but how do we now make it part of the Zig compiler executable?

### Code signature

If you recall, as part of the OS hardening, every executable on arm64/macOS has to be code signed at all times. Both
Apple's system linker `ld64`, and Zig's `zld` will create an adhoc code signature and embed it within the binary by
default. As it turns out, embedding entitlements can be done as part of the said code signature too!

If you are on macOS, you can use `codesign` tool to overwrite the code signature with included entitlements file as
follows

```console
$ codesign --entitlements Info.plist -f -s - myexe
```

This command will replace the existing signature with an adhoc code signature with entitlements from `Info.plist`.
This is great, and that's all that's required to fix the use of `sudo` in our hot-code reloading poc. But, if you are like
me, and you value complete understanding of the entire process involved, it's not enough. So let's try adding the
missing functionality to Zig's linker instead. We will then expose a new flag to Zig's build system which will let
us pass path to entitlements, which can then be routed to the linker for inclusion as part of the generated code
signature. As an added bonus to this approach, besides enhancing our knowledge and understanding, Zig users will not be
required to have command-line tools (CLT) or Xcode installed at all to be able to bake in entitlements in their
executables/dynamic libraries.

First things first though, let's examine the structure of a typical (adhoc) code signature with entitlements. For this,
I will compile a simple _hello world_ style program in C with `zig cc` and replace the signature generated
by `zld` with that generated with `codesign` tool with embedded entitlements. The C program will be dead simple

```c
// hello.c
#include <stdio.h>

int main(int argc, char* argv[]) {
    fprintf(stderr, "Hello, world!\n");
    return 0;
}
```

We will embed _debugging tool entitlement_ depicted in the previous section.

```console
$ zig cc hello.c -o hello
$ codesign --entitlements Info.plist -f -s - hello
hello: replacing existing signature
```

In order to confirm we have properly embedded the entitlement, let's rerun `codesign` in debug-print mode

```console
$ codesign --display --entitlements - hello
Executable=/Users/kubkon/dev/examples/hello_c/hello
[Dict]
        [Key] com.apple.security.cs.debugger
        [Value]
                [Bool] true

```

You can actually use `codesign` to debug-print a lot of information about the embedded code signature, but since we
are trying to explore the precise binary structure of the code signature, I have added verbose printing to a replacement
tool for `otool` I have been writing when learning the Mach-O file format called `zacho` (yeah, I know, I am the king
of the most unimaginative names possible). The source is freely available on GitHub [^4], and here's the output it
produces for `hello`

```console
$ zacho -c hello
```

```
Code signature data:
file = { 49776, 68944 }

{
    Magic = 0xfade0cc0
    Length = 1153
    Count = 5
}
{
    Type: CSSLOT_CODEDIRECTORY(0x0)
    Offset: 52
}
{
    Type: CSSLOT_REQUIREMENTS(0x2)
    Offset: 827
}
{
    Type: CSSLOT_ENTITLEMENTS(0x5)
    Offset: 839
}
{
    Type: CSSLOT_DER_ENTITLEMENTS(0x7)
    Offset: 1093
}
{
    Type: CSSLOT_SIGNATURESLOT(0x10000)
    Offset: 1145
}
{
    Magic: CSMAGIC_CODEDIRECTORY(0xfade0c02)
    Length: 775
    Version: 0x20400
    Flags: 0x2
    Hash offset: 359
    Ident offset: 88
    Number of special slots: 7
    Number of code slots: 13
    Code limit: 49776
    Hash size: 32
    Hash type: 2
    Platform: 0
    Page size: 12
    Reserved: 0
    Scatter offset: 0
    Team offset: 0
    Reserved: 0
    Code limit 64: 0
    Offset of executable segment: 0
    Limit of executable segment: 16384
    Executable segment flags: 0x1

Ident: hello-55554944a708652fc7506bccf6a2b70216280825

Special slot for CSSLOT_DER_ENTITLEMENTS:
        8c0a93d2a721c7b7 8cde6cb284c2e798
        9e9ae0dbe2f85ac1 0fc847f2ca1e0a77

Special slot for CSSLOT_SIGNATURESLOT:
        0000000000000000 0000000000000000
        0000000000000000 0000000000000000

Special slot for CSSLOT_ENTITLEMENTS:
        e8c4dc3360497b20 568c1420ccafec93
        a1511b6b10a9f6da cf9bdfd39e18afed

Special slot for CSSLOT_APPLICATION:
        0000000000000000 0000000000000000
        0000000000000000 0000000000000000

Special slot for CSSLOT_RESOURCEDIR:
        0000000000000000 0000000000000000
        0000000000000000 0000000000000000

Special slot for CSSLOT_REQUIREMENTS:
        987920904eab650e 75788c054aa0b052
        4e6a80bfc71aa32d f8d237a61743f986

Special slot for CSSLOT_INFOSLOT:
        0000000000000000 0000000000000000
        0000000000000000 0000000000000000

Code slot (0x100000000 - 0x100001000):
        b0a6307466b8b198 fb4efed2bf0a035e
        d11954111f2cf856 fb725b145bb91e77

Code slot (0x100001000 - 0x100002000):
        ad7facb2586fc6e9 66c004d7d1d16b02
        4f5805ff7cb47c7a 85dabd8b48892ca7

Code slot (0x100002000 - 0x100003000):
        ad7facb2586fc6e9 66c004d7d1d16b02
        4f5805ff7cb47c7a 85dabd8b48892ca7

Code slot (0x100003000 - 0x100004000):
        110721bd92675d0f 0641b8963afc09f8
        45d34efd762677c6 a4a3bb222fa75562

Code slot (0x100004000 - 0x100005000):
        ad7facb2586fc6e9 66c004d7d1d16b02
        4f5805ff7cb47c7a 85dabd8b48892ca7

Code slot (0x100005000 - 0x100006000):
        ad7facb2586fc6e9 66c004d7d1d16b02
        4f5805ff7cb47c7a 85dabd8b48892ca7

Code slot (0x100006000 - 0x100007000):
        ad7facb2586fc6e9 66c004d7d1d16b02
        4f5805ff7cb47c7a 85dabd8b48892ca7

Code slot (0x100007000 - 0x100008000):
        ad7facb2586fc6e9 66c004d7d1d16b02
        4f5805ff7cb47c7a 85dabd8b48892ca7

Code slot (0x100008000 - 0x100009000):
        5ab20475893bf05f 3e1078c8270f0efc
        82319c5257e92cea 05f969d3eaeb2d4c

Code slot (0x100009000 - 0x10000a000):
        ad7facb2586fc6e9 66c004d7d1d16b02
        4f5805ff7cb47c7a 85dabd8b48892ca7

Code slot (0x10000a000 - 0x10000b000):
        ad7facb2586fc6e9 66c004d7d1d16b02
        4f5805ff7cb47c7a 85dabd8b48892ca7

Code slot (0x10000b000 - 0x10000c000):
        ad7facb2586fc6e9 66c004d7d1d16b02
        4f5805ff7cb47c7a 85dabd8b48892ca7

Code slot (0x10000c000 - 0x10000d000):
        46fa859070a7a3b8 232b701b96750232
        c34d6c25ed775969 a90932f3417526dc
}
{
    Magic: CSMAGIC_REQUIREMENTS(0xfade0c01)
    Length: 12
    Data:
        0000000000000000 0000000000000000  \x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00
}
{
    Magic: CSMAGIC_EMBEDDED_ENTITLEMENTS(0xfade7171)
    Length: 254
    Data:
        3c3f786d6c207665 7273696f6e3d2231  <?xml version="1
        2e302220656e636f 64696e673d225554  .0" encoding="UT
        462d38223f3e0a3c 21444f4354595045  F-8"?>\x0a<!DOCTYPE
        20706c6973742050 55424c494320222d   plist PUBLIC "-
        2f2f4170706c652f 2f44544420504c49  //Apple//DTD PLI
        535420312e302f2f 454e222022687474  ST 1.0//EN" "htt
        703a2f2f7777772e 6170706c652e636f  p://www.apple.co
        6d2f445444732f50 726f70657274794c  m/DTDs/PropertyL
        6973742d312e302e 647464223e0a3c70  ist-1.0.dtd">\x0a<p
        6c69737420766572 73696f6e3d22312e  list version="1.
        30223e0a3c646963 743e0a202020203c  0">\x0a<dict>\x0a    <
        6b65793e636f6d2e 6170706c652e7365  key>com.apple.se
        6375726974792e63 732e646562756767  curity.cs.debugg
        65723c2f6b65793e 0a202020203c7472  er</key>\x0a    <tr
        75652f3e0a3c2f64 6963743e0a3c2f70  ue/>\x0a</dict>\x0a</p
        6c6973743e0a0000 0000000000000000  list>\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00
}
{
    Magic: CSMAGIC_EMBEDDED_DER_ENTITLEMENTS(0xfade7172)
    Length: 52
    Data:
        702a020101b02530 230c1e636f6d2e61  p*\x02\x01\x01\xb0%0#\x0c\x1ecom.a
        70706c652e736563 75726974792e6373  pple.security.cs
        2e64656275676765 720101ff00000000  .debugger\x01\x01\xff\x00\x00\x00\x00
}
{
    Magic: CSMAGIC_BLOBWRAPPER(0xfade0b01)
    Length: 8
    Data:
}
```
(_NB the zero-padding you see in `zacho`'s output is just an artifact of how it prints the binary blobs. The actual
contents of the code signature structure does not require any alignment between its different internal blobs._)

Hmm, OK, this is quite a lot... Let's start from the beginning then. Each code signature starts with a header called
`SuperBlob`

```zig
const CSMAGIC_EMBEDDED_SIGNATURE: u32 = 0xfade0cc0;

const SuperBlob = extern struct {
    /// Magic number
    magic: u32 = CSMAGIC_EMBEDDED_SIGNATURE,

    /// Total length of SuperBlob
    length: u32,

    /// Number of index BlobIndex entries following this struct
    count: u32,
};
```

(_NB almost forgot to mention that every field in the code signature is in big-endian, which if forgotten about, can
render your code signing tool absolutely useless._)

For our purposes, `magic` will always equal `CSMAGIC_EMBEDDED_SIGNATURE` but, in general, as far as I can tell, this
does not have to be the case. The `length` field denotes the total length of the code signature (with all its bells
and whistles), while `count` denotes how many `BlobIndex`es are following it. Here is the definition of a `BlobIndex`

```zig
const BlobIndex = extern struct {
    /// Type of entry
    @"type": u32,

    /// Offset of entry
    offset: u32,
};
```

As you probably have guessed by now, each `BlobIndex` tells you at which offset in the code signature an inner blob of
some type is embedded such as the blob storing the adhoc signature, etc. In our example, we have a total count of 5
inner blobs: code directory (`CSSLOT_CODEDIRECTORY`), requirements (`CSSLOT_REQUIREMENTS`), entitlements
(`CSSLOT_ENTITLEMENTS`), Distinguished Encoding Rules (DER) entitlements (`CSSLOT_DER_ENTITLEMENTS`), and signature
(`CSSLOT_SIGNATURESLOT`).

The parser can use `BlobIndex` to selectively jump to the desired embedded blob without having to parse the entire
code signature. Each embedded blob shares a common header called `GenericBlob` which is defined as

```zig
const GenericBlob = extern struct {
    /// Magic number
    magic: u32,

    /// Total length of blob
    length: u32,

    /// Followed by a slice of bytes of length `length - 8`
    data: [*]u8,
};
```

Clearly, `GenericBlob` acts as a generic header structure for some actual blob that is distinguished by its
magic number. Currently, only code directory has a specialised blob structure. Every other blob is represented by
a `GenericBlob`.

In what follows, I'd like us to focus on code directory (`CSMAGIC_CODEDIRECTORY`) and entitlements (`CSMAGIC_EMBEDDED_ENTITLEMENTS`)
blobs. This is all we need to embed the entitlements within a code signature actually; none of the extras generated
by `codesign` tool are actually needed. It's worth pointing out that the adhoc code signature as generated by the linkers
consists of only the former, i.e., the code directory.

#### Code directory

The code directory structure comprises of many fields

```zig
const CodeDirectory = extern struct {
    /// Magic number (CSMAGIC_CODEDIRECTORY)
    magic: u32,

    /// Total length of CodeDirectory blob
    length: u32,

    /// Compatibility version
    version: u32,

    /// Setup and mode flags
    flags: u32,

    /// Offset of hash slot element at index zero
    hashOffset: u32,

    /// Offset of identifier string
    identOffset: u32,

    /// Number of special hash slots
    nSpecialSlots: u32,

    /// Number of ordinary (code) hash slots
    nCodeSlots: u32,

    /// Limit to main image signature range
    codeLimit: u32,

    /// Size of each hash in bytes
    hashSize: u8,

    /// Type of hash (cdHashType* constants)
    hashType: u8,

    /// Platform identifier; zero if not platform binary
    platform: u8,

    /// log2(page size in bytes); 0 => infinite
    pageSize: u8,

    spare2: u32,
    scatterOffset: u32,
    teamOffset: u32,
    spare3: u32,
    codeLimit64: u64,

    /// Offset of executable segment
    execSegBase: u64,

    /// Limit of executable segment
    execSegLimit: u64,

    /// Executable segment flags
    execSegFlags: u64,
};
```

However, of particular interest are `hashOffset`, `identOffset`, `nSpecialSlots`, and `nCodeSlots`.
In order to describe each of the fields, let's first examine how a typical code directory blob is laid out in file.
Following its header, we expect a `NUL`-delimited C string containing the identifier of the program. The first element of this
string coincides with the offset stored as `identOffset` in the header. The length of the string is not stored since it's
`NUL`-delimited. In our example, the identifier equals `hello-55554944a708652fc7506bccf6a2b70216280825`.

```
{
    Magic: CSMAGIC_CODEDIRECTORY(0xfade0c02)
    Length: 775
    Version: 0x20400
    Flags: 0x2
    Hash offset: 359
    Ident offset: 88
    Number of special slots: 7
    Number of code slots: 13
    Code limit: 49776
    Hash size: 32
    Hash type: 2
    Platform: 0
    Page size: 12
    Reserved: 0
    Scatter offset: 0
    Team offset: 0
    Reserved: 0
    Code limit 64: 0
    Offset of executable segment: 0
    Limit of executable segment: 16384
    Executable segment flags: 0x1

Ident: hello-55554944a708652fc7506bccf6a2b70216280825

...
```

Next, we expect `N := nSpecialSlots` hashes of size `hashSize` and type `hashType` (e.g., `32` bytes long and of type
`Sha256`). OK, but what are we actually hashing? Special slots encode hashes of other embedded blobs such as entitlements.
A couple important points to note here. The number of special slots has a static maximum of `7`, however, not all slots
need to be present (if you run `zacho` on a binary produced by `ld64` or `zld`, since the linkers only embed adhoc
code signature by default, you will notice there are no special slots present). Starting index is `#1` rather than
`#0` as we are normally used to. The slots have their indices in decreasing order, i.e., slot `#7` is written to file
first, then `#6`, etc. Indexes for each slot have a fixed meaning represented by the following constants

```zig
const CSSLOT_INFOSLOT: u32 = 1;
const CSSLOT_REQUIREMENTS: u32 = 2;
const CSSLOT_RESOURCEDIR: u32 = 3;
const CSSLOT_APPLICATION: u32 = 4;
const CSSLOT_ENTITLEMENTS: u32 = 5;
const CSSLOT_DER_ENTITLEMENTS: u32 = 7;
```

with one exception: index `#6` encodes a signature slot which is represented by a constant `const CSSLOT_SIGNATURESLOT: u32 = 0x10000`.
When populating slots, if we populate entitlements slot which happens at index `#5`, we need to write out every
other slot with index smaller than `#5`. If those other slots are unpopulated in the code signature, we simply write
out hashes filled with `hashSize` of `0`s.

```
...

Special slot for CSSLOT_DER_ENTITLEMENTS:
        8c0a93d2a721c7b7 8cde6cb284c2e798
        9e9ae0dbe2f85ac1 0fc847f2ca1e0a77

Special slot for CSSLOT_SIGNATURESLOT:
        0000000000000000 0000000000000000
        0000000000000000 0000000000000000

Special slot for CSSLOT_ENTITLEMENTS:
        e8c4dc3360497b20 568c1420ccafec93
        a1511b6b10a9f6da cf9bdfd39e18afed

Special slot for CSSLOT_APPLICATION:
        0000000000000000 0000000000000000
        0000000000000000 0000000000000000

Special slot for CSSLOT_RESOURCEDIR:
        0000000000000000 0000000000000000
        0000000000000000 0000000000000000

Special slot for CSSLOT_REQUIREMENTS:
        987920904eab650e 75788c054aa0b052
        4e6a80bfc71aa32d f8d237a61743f986

Special slot for CSSLOT_INFOSLOT:
        0000000000000000 0000000000000000
        0000000000000000 0000000000000000

...
```

Finally, following the special slots, we should reach offset of `hashOffset` which denotes the start of code hashes:
hashes of the executable file split into pages of size `pageSize`. Note that the `pageSize` does not have to be equal
to the actual hardware page size (e.g., `16KB` for arm64), and is typically set to `4KB` by Apple tooling.
`zld` sets it according to the hardware page size.

```
...

Code slot (0x100000000 - 0x100001000):
        b0a6307466b8b198 fb4efed2bf0a035e
        d11954111f2cf856 fb725b145bb91e77

Code slot (0x100001000 - 0x100002000):
        ad7facb2586fc6e9 66c004d7d1d16b02
        4f5805ff7cb47c7a 85dabd8b48892ca7

...

Code slot (0x10000c000 - 0x10000d000):
        46fa859070a7a3b8 232b701b96750232
        c34d6c25ed775969 a90932f3417526dc
}
```

#### Embedded entitlements

If you recall, embedded entitlements are encoded using `GenericBlob` directly where the contents of the entitlements
file is embedded as raw bytes in the `data` field

```
{
    Magic: CSMAGIC_EMBEDDED_ENTITLEMENTS(0xfade7171)
    Length: 254
    Data:
        3c3f786d6c207665 7273696f6e3d2231  <?xml version="1
        2e302220656e636f 64696e673d225554  .0" encoding="UT
        462d38223f3e0a3c 21444f4354595045  F-8"?>\x0a<!DOCTYPE
        20706c6973742050 55424c494320222d   plist PUBLIC "-
        2f2f4170706c652f 2f44544420504c49  //Apple//DTD PLI
        535420312e302f2f 454e222022687474  ST 1.0//EN" "htt
        703a2f2f7777772e 6170706c652e636f  p://www.apple.co
        6d2f445444732f50 726f70657274794c  m/DTDs/PropertyL
        6973742d312e302e 647464223e0a3c70  ist-1.0.dtd">\x0a<p
        6c69737420766572 73696f6e3d22312e  list version="1.
        30223e0a3c646963 743e0a202020203c  0">\x0a<dict>\x0a    <
        6b65793e636f6d2e 6170706c652e7365  key>com.apple.se
        6375726974792e63 732e646562756767  curity.cs.debugg
        65723c2f6b65793e 0a202020203c7472  er</key>\x0a    <tr
        75652f3e0a3c2f64 6963743e0a3c2f70  ue/>\x0a</dict>\x0a</p
        6c6973743e0a0000 0000000000000000  list>\x0a\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00
}
```

TODO

### Hot-code reloading revisited demo - no sudo!

_Demo captured on M1 MacBook Air, macOS 12.2.1, latest Zig self-hosted compiler with patch from `hcs-macos` branch._ [^6]

### References

[^1]: [Stock `lldb`'s sources'](https://github.com/llvm/llvm-project/tree/main/lldb)
[^2]: [Entitlements - Apple's official documentation](https://developer.apple.com/documentation/bundleresources/entitlements)
[^3]: [Debugging Tool Entitlement - Apple's official documentation](https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_security_cs_debugger)
[^4]: [Zacho - Zig's Mach-O parser](https://github.com/kubkon/zacho)
[^5]: [zig-snapshots - a tool for previewing snapshots of Zig's incremental linker states](https://github.com/kubkon/zig-snapshots)
[^6]: [Git diff of the changes to the Zig compiler to enable hot-code reloading on macOS](https://github.com/ziglang/zig/compare/hcs-macos)
[^7]: ["Mac OS X and Task_for_pid() Mach Call" by Ivan Ostres](http://os-tres.net/blog/2010/02/17/mac-os-x-and-task-for-pid-mach-call/)
[^8]: [`lldb/debugserver` for macOS](https://github.com/llvm/llvm-project/tree/main/lldb/tools/debugserver/source/MacOSX)
