---
layout: post
title: 'Porting projects to WASI: the flite program'
---

# {{ page.title }}

This post is hopefully the first of many more to come demonstrating my efforts
to port as many advanced projects written in C/C++ to WASI as possible. What is
WASI? WASI is a system interface to run WebAssembly outside the web. If you would
like to learn more, I cannot recommend enough the fantastic blog post by Lin Clark
[Standardising WASI](https://hacks.mozilla.org/2019/03/standardizing-wasi-a-webassembly-system-interface/).
You should definitely go and check it out!

We'll start our journey with a fantastic, open-source text-to-speech program
called [flite](https://github.com/festvox/flite). Without further ado, let's dig in!

## Getting the right tools
Firstly, make sure you've got all the right tools for the job. In this case, I mean
the WASI SDK which you can get from [CraneStation/wasi-sdk](https://github.com/CraneStation/wasi-sdk).
The SDK features `clang-8` which is capable of targetting `wasm32-unknown-wasi` triple plus
it also features the WASI sysroot. So head over to the website, and get the SDK.

When porting any C program to WASI, there are a few things to watch out for. Firstly,
WASI currently doesn't support `setjmp` nor sockets so the source code will need to modified
accordingly (see [here](https://github.com/CraneStation/wasi-sysroot/tree/master/libc-top-half)
for missing functionality in WASI). In case of `flite`, it's somewhat easier
as there are two macros `DIE_ON_ERROR`
and `CST_NO_SOCKETS` which pretty much handle most it for us.

Secondly, remember to correctly set the paths to the compiler, `llvm-ar`, `llvm-ranlib`, etc. That is, don't trust
`configure` to get it right (since WASI is still experimental, the tools won't be included
as OS packages any time soon). Otherwise, you are almost guaranteed to experience missing symbols
during linking or other weird errors.

Finally, if the project is using `configure`, you'll need to take extra care at including WASI as a 
supported host. A brilliant, short-and-simple version
[suggested by Frank Denis](https://00f.net/2019/04/07/compiling-to-webassembly-with-llvm-and-clang/)
is using `grep` and `sed`:

{% highlight console %}
$ grep -q -F -- '-wasi' config.sub || sed -i -e 's/-nacl\*)/-nacl*|-wasi)/' config.sub
{% endhighlight %}

## Porting flite
I've modified the source of `flite` to make it easier to port it to WASI. The source code can be cloned
from [kubkon/flite](https://github.com/kubkon/flite). Furthermore, if you're interested in the changes
I've had to introduce to the original project, simply compare the fork against the original in Github.

Thus, let's clone it

{% highlight console %}
$ git clone https://github.com/kubkon/flite
{% endhighlight %}

At this point, you need to have the WASI SDK installed. If you don't have it yet, head over to
[CraneStation/wasi-sdk](https://github.com/CraneStation/wasi-sdk) and install it. For the rest
of this blog post, I'll assume that you have the SDK installed in `/opt/wasi-sdk`.

Let's spin `configure` and `make` then!

{% highlight console %}
$ ./configure --host=wasm32-unknown-wasi CC="/opt/wasi-sdk/bin/clang --sysroot=/opt/wasi-sdk/share/sysroot" AR=/opt/wasi-sdk/bin/llvm-ar RANLIB=/opt/wasi-sdk/bin/llvm-ranlib
$ make
{% endhighlight %}

If everything went well, you should now have `bin/flite` program compiled to WASI, and you should be able
to run it in any WASI compatible runtime such as [CraneStation/wasmtime](https://github.com/CraneStation/wasmtime):

{% highlight console %}
$ wasmtime bin/flite -- --help
flite: a small simple speech synthesizer
  Carnegie Mellon University, Copyright (c) 1999-2016, all rights reserved
  version: flite-2.2-current Sep 2018 (http://cmuflite.org)
usage: flite TEXT/FILE [WAVEFILE]
  Converts text in TEXTFILE to a waveform in WAVEFILE
  If text contains a space the it is treated as a literal
  textstring and spoken, and not as a file name
  if WAVEFILE is unspecified or "play" the result is
  played on the current systems audio device.  If WAVEFILE
  is "none" the waveform is discarded (good for benchmarking)
  Other options must appear before these options
  --version   Output flite version number
  --help      Output usage string
  -o WAVEFILE Explicitly set output filename
  -f TEXTFILE Explicitly set input filename
  -t TEXT     Explicitly set input textstring
  -p PHONES   Explicitly set input textstring and synthesize as phones
  --set F=V   Set feature (guesses type)
  -s F=V      Set feature (guesses type)
  --seti F=V  Set int feature
  --setf F=V  Set float feature
  --sets F=V  Set string feature
  -ssml       Read input text/file in ssml mode
  -b          Benchmark mode
  -l          Loop endlessly
  -voice NAME Use voice NAME (NAME can be pathname/url to flitevox file)
  -voicedir NAME Directory containing (clunit) voice data
  -lv         List voices available
  -add_lex FILENAME add lex addenda from FILENAME
  -pw         Print words
  -ps         Print segments
  -psdur      Print segments and their durations (end-time)
  -pr RelName Print relation RelName
  -voicedump FILENAME Dump selected (cg) voice to FILENAME
  -v          Verbose mode
$ wasmtime --dir=. bin/flite -- test.txt test.wav
$ file test.wav
test.wav: RIFF (little-endian) data, WAVE audio, Microsoft PCM, 16 bit, mono 8000 Hz
{% endhighlight %}

I hope this was useful for you! If you have any questions, comments or suggestions, feel free to drop me a line!
