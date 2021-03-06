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
retrieve where to look next, also inline annotations for julia code include function definition in source files. In our example

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
As an alternative we can use `perf diff`, a tool that by default confronts `perf.data` and `perf.data.old` but can look up a variety of baseline input files (see [man pages](https://linux.die.net/man/1/perf-diff) ).

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

....................
#
11.49%     -3.91%  libjulia.so.0.7.0    [.] jl_read_relocations
0.73%     +1.66%  libjulia.so.0.7.0    [.] jl_apply_generic
0.29%     +1.55%  libjulia.so.0.7.0    [.] obviously_disjoint.part.15
0.95%     +1.44%  libjulia.so.0.7.0    [.] forall_exists_subtype
3.66%     -1.21%  [unknown]            [k] 0xffffffffb580015f
2.77%     -1.03%  libpthread-2.27.so   [.] __pthread_mutex_lock
0.36%     +0.97%  libjulia.so.0.7.0    [.] jl_alloc_array_1d
0.08%     +0.93%  sys.so               [.] japi1_getindex_1361
0.90%     +0.90%  libjulia.so.0.7.0    [.] subtype
:

```
In general on the left we have the baseline value, on the right the difference in newer file.
The difference in `jl_read_relocations` can be imputed to the fact that the second script spends
less time on runtime precompilation.

While thread synchronization is faster, due to the nature of broadcasting,
more time is spent instead on dispatch, subtyping and indexing.

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

Run
```
$ perf c2c record --call-graph dwarf,8192 -F 60000 -a sleep 5
$ ./julia /tmp/cache.jl
$ perf c2c report -NN -g --call-graph -c pid,iaddr --stdio
```
Unofrtunately tracing a single compilation leads to a segfault ([kernel bugzilla](https://bugzilla.kernel.org/show_bug.cgi?id=200613).
At the moment is meant as a system wide only tracing tool,
so you will need to run Julia while tracing with perf c2c.
The first thing to keep an eye on is HITM, which are hits in a modified cacheline, divided in local and remote nodes.
In the second table present in the output you should find the most used cachelines.

```
=================================================
            Trace Event Information
=================================================
  Total records                     :       1688
  Locked Load/Store Operations      :         95
  Load Operations                   :        540
  Loads - uncacheable               :          0
  Loads - IO                        :          0
  Loads - Miss                      :         19
  Loads - no mapping                :         19
  Load Fill Buffer Hit              :        117
  Load L1D hit                      :        157
  Load L2D hit                      :          7
  Load LLC hit                      :         70
  Load Local HITM                   :          0
  Load Remote HITM                  :          0
  Load Remote HIT                   :          0
  Load Local DRAM                   :        151
  Load Remote DRAM                  :          0
  Load MESI State Exclusive         :        151
  Load MESI State Shared            :          0
  Load LLC Misses                   :        151
  LLC Misses to Local DRAM          :      100.0%
  LLC Misses to Remote DRAM         :        0.0%
  LLC Misses to Remote cache (HIT)  :        0.0%
  LLC Misses to Remote cache (HITM) :        0.0%
  Store Operations                  :       1148
  Store - uncacheable               :          0
  Store - no mapping                :          0
  Store L1D Hit                     :       1117
  Store L1D Miss                    :         31
  No Page Map Rejects               :        143
  Unable to parse data source       :          0

```

Please consider that first the `-F` flag argument must be tuned on the frequency allowed on your kernel. Also always precise that the debug info in `--call-graph` is in `dwarf` format or you may run in a segmentation fault.

For a more detailed guide on c2c usage, please refer to [joe mario's blog](https://joemario.github.io/blog/2016/09/01/c2c-blog/).

## Visualization of data
Even considering tricks, `perf` data can still be harsh to digest. That's why many visualization tools have been developed to give a better prespective.
Personally I have been able to use [gprof2dot](https://github.com/jrfonseca/gprof2dot), unfortunately I have not been to use Google's `perf_data_converter` due to [an issue](https://github.com/google/perf_data_converter/issues/40) that seems to be unrelated to Julia integration in particular.
In conclusion, I advise you to look up [Brendan Regg's perf examples](www.brendangregg.com/perf.html) for tutorials on other visualization tools like FlameGraphs.
