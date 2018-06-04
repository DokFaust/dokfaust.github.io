
---
layout: post
title: "Gsoc week 3 report"
tags: [ gsoc, julia, llvm, perf, report ]
---

# External Profiling in Julia

To begin my coding pediod I had to reintegrate three external profiling
tools : __perf__, __OProfile__ and __Intel Vtune__. In LLVM external
profiling tools interfaces are managed as JITEventListener classes.

Unfortunately LLVM ORC JIT does not support natively [JITEventListener](https://groups.google.com/forum/#!topic/llvm-dev/B7quHkDRoYk), but on the other hand, a way of registering objects and stacking them locally is present in the Julia JIT engine under the subclass `debugobjregistrar`.

At this point it was necessary for me to implement just `RegisterJITEventListener`, 
which managaed a vector of `JITEventListener`s, and `NotifyFinalized`, 
which for each EventListener calls the `NotifyObject` emitted method.

Notifying an object usually means dumping the execution details in a file
that can be parsed by the profiling tool.

Then for each external profiling tool, I went on to hook it up to the Julia
JIT Execution Engine.

## Intel VTune

This was the easy one. The LLVM wrapper includes by default line-level
annotations, which seem to be managed by jit profiling. In this case the
LLVM wrapper was working as it is, so I avoided to move it inside the Julia
codebase.

## Perf

As Perf by itself is not implemented in LLVM, the [D44892](https://reviews.llvm.org/D44892) has been very helpful.
At the beginning I had the problem of understanding how to implement
RegisterJITEventListener for the JuliaOJIT object, but my mentors where
very helpful in this.

Now the Perf module is working, except that Julia user defined functions 
are registered after they are mangled, which makes it more difficult to 
understand `perf` reports.

As a note for future work, my mentor noticed that the `@profile`
macro could be implemented to use `perf` externally. This is something
where I can work on, also if it is intended as a tool for timing and cpu
perfomance it won't need to interface with the JIT compilation directly, as
done in [LinuxPerf.jl](https://github.com/DokFaust/LinuxPerf.jl/).

## OProfile 

This was a bit tough, even if it did not required so much work.
The implementation of the OProfile wrapper can be found in LLVM, also
OProfile currently offers in the wrapper an interface to dump executed code
in a file. [ref](http://oprofile.sourceforge.net/doc/devel/jit-interface.html)

I had some problems at first with the event agent implementation, as
during basecompiler.ji compilation `make` segfaulted.

It became obvious that the `AgentWrapper` object had the wrong lifetime, as
when debugging with `gdb` the compiler received __SIGSEGV__ while executing
`NotifyObjectEmitted` in the Oprofile module.

Than with __rr__ i was able to debug the error backward, where I noticed
that `Wrapper->isAgentAvailable`, (ie check if agent is instatiated), 
was pointing to `0x00`, that made it obvious where the bug originated.
`rr` backward execution here saved me many hours or even days!

Now the OProfile module is working, the only problem that I am still facing
is in line-level annotations.

### DWARF debug info integration in OProfile

OProfile wrapper uses the `op_write_native_code` function to dump object
files. For line-level and code annotations it relies on
`op_write_debug_line_info` which accepts as input a pointer to the code
location, an array of `op_write_debug_line_info` entries and the number of entries.

The problem is that LLVM does not implement line-level annotations for
OProfile, which if we want to integrate that in Julia should be done
manually. 
That's why I included an external module in the Julia source,
after it is ready we can produce an LLVM patch and include just that.

[`op_write_debug_line_info`](http://oprofile.sourceforge.net/doc/devel/op_write_debug_line_info.html) is composed of three fields : vma, line number and
filename. `vma` in the docs is either defined as an [unsigned long](https://www.cs.rice.edu/~la5/doc/oprofile/d0/d27/structdebug__line__info.html#a9d5640d6fea5ff12a5545d8f8940424c) which represents the starting address of the profiling interface, which in the case of the JVM is [JVMTI](http://www.oracle.com/technetwork/articles/javase/jvmti-136367.html).
As an implementation of `op_write_debug_line_info` I have found only the
one present in the [JVM source](https://github.com/aosp-mirror/platform_external_oprofile/blob/3722f1053f4cab90c4daf61451713a2d61d79c71/agents/jvmti/libjvmti_oprofile.c) which seems a bit problematic to implement, especially for how to construct the `debug_line_info` struct using the __DWARF__ context built from JITed objects.

# Other activities related to this working period
 
* For thread-safe RNG I have tried to test the [KissThreading.jl](https://github.com/bkamins/KissThreading.jl/blob/master/src/KissThreading.jl#L7-L13) implementation which effectively seems to give a slowdown, especially when running eith one thread.
  I will check if implementing similalry a thread-local array of
  Mersennetwister RNGs using `task_local_storage` gives a sensible speedup.
  Also I will look if it is better to define a wrapper method that checks if
  Julia is running on a single thread, in that case the ordinary `rand()`
  definition is provided.

* As a warmup before working with RNG I have studied a bit the
  implementation in some working cases. A blog post will come lately.

* While working with profiling tools, I finally started to work with the
  radare r2k module, that interfaces radare with the linux kernel. It is a
  very powerful tool to trace race conditions on the linux kernel.
  If I have some sparetime left, I will write a blog entry. Stay Tuned!

