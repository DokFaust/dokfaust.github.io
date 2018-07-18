
---
layout: post
title: "Profiling Julia code with perf!"
tags: [ gsoc, julia, perf, guide ]
---

# Performance analysis on Julia code.

Lately we lost the capability of using external profiling tools with Julia. This was due to the passage from MCJIT to ORCJIT [(ref issue 26999)](https://github.com/JuliaLang/julia/issues/26999)
in LLVM JIT execution engine, which lost object notification to profilers at runtime. This was implemented manually in [PR #27466](https://github.com/JuliaLang/julia/pull/27466), where each of
the external profiling tools that are supported in LLVM (Oprofile, Vtune, Perf) is notified separately. Today we will concentrate on Linux Perf, as it is more widespread and the newest adoption in LLVM codegen.

## What is perf? And why should you use it?
In Linux userland we have different tracing tools that we can choose from. One of the key aspects of `Perf` is that it presents tools both for userspace events and kernelspace (`perf_event_open` syscall).
This presents us with events of different kinds

1) *Software events* : context switches, minor faults.
2) *Hardware events* : number of cycles, L1 chache misses and so on.
3) *Tracepoint events* : implemented through the kernel `ftrace` infrastructure, include scheduler events and syscall monitors, which can target sockets entrance\exit ecc.

Even tough most of this won't come in use in everyday development, it will still be useful in particular occasions

This allows us to get different metrics of the performance of our Julia code. If you are running on Windows/MacOSX system, please consider using Intel VTune.

## Sample how-to on running Julia code

Compile Julia with `USE_PERF_JITEVENTS=1` in `Make.user` and be sure that the environment variable `ENABLE_JITPROFILING` is on (ie `export ENABLE_JITPROFILING=1`).
Now JIT events will be transcribed to a file present in `/tmp/PID.map`. Let's work on a simple example : consider the code sample below

```
function naive_isprime(n::Int64)
    if (n==2)
        return true
    elseif (n==1 || n % 2 == 0)
        return false
    else
        for i in 3:2:n
            if (n % i == 0)
                return false
            end
        end
    end
    return true
end

global num_primes = 0

for i in 1:10000
    if (naive_isprime(i))
        num_primes+=1
    end
end
```
We are interested in drowning CPU resources. We run
```
$ perf record -o /tmp/perf.data --call-graph dwarf ./julia /tmp/stupidprime.jl
$ perf report --call-graph -G
```
With this in mind, the perf output can look quite messy. In general looking at generated code should not be a great idea for a smaller use case, but sometimes code annotations can help us
retrieve where to look next, also inline annotations for julia code include also function definition in files. In our example

```
./compiler/optimize.jl:110
    │       mov    0x10(%rcx),%r15
│       mov    %r15,0x50(%rbx)
    │     _growend!():
        │     ./array.jl:789
        │       mov    $0x1,%esi
        │       mov    %r15,%rdi
        │     → callq  jlplt_jl_array_grow_end_157_got
        │     length():
            │     ./array.jl:174
            │       mov    0x8(%r15),%r14
            │     push!():
                │     ./array.jl:838
```
## The bad and the good one : using perf diff

Generally we may be interested in working on bugged examples where we need to check, after a small code revision, why we get a performance regression.
Looking to that from a full `perf record` can be quite a waste of time.
As an alternative we can use `perf diff`, a tool that by default confronts `perf.data` and `perf.data.old` but can look up a variety of baseline input files (see [man pages](https://linux.die.net/man/1/perf-diff).
Consider for example [issue 28126](https://github.com/JuliaLang/julia/issues/28126) where the performance of broadcasting vs. a for loop is analized in Julia.
For simplicity I saved the snippets in two separete files, `broadcasting.jl` and `for.jl`. Record each of them separately :
```
$ perf record --call-graph dwarf ./julia /tmp/for.jl
$ perf record --call-graph dwarf ./julia /tmp/broadcasting.jl
```
Notice that as broadcasting is slower, we first measure the for loop. In this way it will be the "old" file that is considered as baseline.
As the two files `perf.data` and `perf.data.old` are present, you can go on
```
$ perf diff
```
It is immediate to notice that a greater percentage of time is spent on `jl_read_relocations` which is probably due to more time needed to restore the previous system image.
Also it spends less time on `jl_apply_generic`, where depending on the context, it can be wrapping around a closure.

## Other utilities : c2c, dynamic and static probing

You can consider, if you need to target a specific event that is manifested system-wide (eg a particular syscalls, page-faults ecc.) dynamic and static probing. This is independent of Julia integration,
but if you need inspiration you look at [Brendan Regg's perf examples](www.brendangregg.com/perf.html).

Another useful tool is `perf c2c` which is perf's utility for multi-thread, multi-process and NUMA application's, mostly used to detect [false sharing](https://software.intel.com/en-us/articles/avoiding-and-identifying-false-sharing-among-threads).

Consider the code below, take from [Valentin Churavy's notes](http://slides.com/valentinchuravy/julia-parallelism#/5/8),save it in `/tmp/cache.jl` and run on terminal `$ export JULIA_NUM_THREADS=4`.
The function `f()` is susceptible to false-sharing, while `g()` is buffered to avoid that threads access contiguous areas of memory.

```
using Base.Threads

function f()
    acc = zeros(Int64, nthreads())
    @threads for tid in 1:nthreads()
        for i in 1:10000
            acc[tid] += 1
        end
    end
    sum(acc)
end

#buffered version

function g()
    CACHE_LINE = 64
    elems = div(CACHE_LINE, sizeof(int64))
    acc = zeros(Int64, nthreads() * elems)
    @threads for tid in 1:nthreads()
        store = (tid-1)*elems +1
        for i in 1:10000
            acc[store] += 1
        end
    end
    sum(acc)
end

f()
```

Run `perf c2c record ./julia /tmp/cache.jl`, and then `perf c2c report`. The first thing to keep an eye on is HITM, which are hits in a modified cacheline, divided in local and remote nodes.
In the second table present in the output you should find the most used cachelines. For a more detailed guide on c2c usage, please refer to [joe mario's blog](https://joemario.github.io/blog/2016/09/01/c2c-blog/).

## Visualization of data
Even considering tricks, `perf` data can still be harsh to digest. That's why many visualization tools have been developed to give a better prespective.
Personally I have been able to use [gprof2dot](https://github.com/jrfonseca/gprof2dot), unfortunately I have not been to use Google's `perf_data_converter` due to [an issue](https://github.com/google/perf_data_converter/issues/40) that seems to be unrelated to Julia integration in particular.
In conclusion, I advise you to look up [Brendan Regg's perf examples](www.brendangregg.com/perf.html) for tutorials on other visualization tools like FlameGraphs.
