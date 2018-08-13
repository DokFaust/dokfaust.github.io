---
layout: post
title: "Gsoc week 3 report"
tags: [ gsoc, julia, llvm, perf, report, RNG, RCU ]
---

# Thread-Safety in Julia : a survey of my project

For the GSoC of 2018 my project "Thread-Safety in the Julia compiler" was accepted
by NumFocus. Jameson Nash, Valentin Churavy and Christopher Rackauckas were my mentors.
As a reference, there is the [Project Idea's Page](https://julialang.org/soc/projects/compiler.html) and the [final proposal](https://docs.google.com/document/d/1n0v_RueP4F3qV88RARSNBt4AHmV_9u4ONvGvHW2qC2U/edit).

The project can be divided into three main subtopics :

1) Reintegrate external profiling in the Julia JIT with OProfile and Intel VTune, and port perf too.

2) Implement a `rand()` method that returns an indipendent result between threads.

3) Work on typemap cache store/load synchronization between threads by implementing RCU primitives.

## Reintegrate external profiling

In LLVM external profiling tools interfaces are managed as JITEventListener classes.
To keep up with the LLVM upstream changes, the Julia JIT was ported to ORCJIT.

Unfortunately LLVM ORC JIT does not support natively [JITEventListener](https://groups.google.com/forum/#!topic/llvm-dev/B7quHkDRoYk), but on the other hand, a way of registering objects and stacking them locally is present in the Julia JIT engine under the subclass `DebugObjectRegistrar`.
This

At this point it was necessary for me to implement just `RegisterJITEventListener`,
which managed a vector of `JITEventListener`s, and `NotifyFinalized`,
which for each EventListener calls the `NotifyObject` emitted method.

Notifying an object usually means dumping the execution details in a file
that can be parsed by the profiling tool.

Then for each external profiling tool, I went on to hook it up to the Julia
JIT Execution Engine.

The core of it is in Julia JIT `NofityFinalzer` implementation

```
void JuliaOJIT::NotifyFinalizer(const object::ObjectFile &Obj,
                                const RuntimeDyld::LoadedObjectInfo &LoadedObjectInfo)
{
	for (auto &Listener : EventListeners)
		Listener->NotifyObjectEmitted(Obj, LoadedObjectInfo);
}
```

You can checkout the status of the PR on [Github](https://github.com/JuliaLang/julia/pull/27466).

### Intel VTune

This was the easy one. The LLVM wrapper includes by default line-level
annotations, which seem to be managed by jit profiling.

### Perf

As Perf notification is not supported in LLVM, the [D44892](https://reviews.llvm.org/D44892) LLVM review has proved to be necessary for the scope. It has been upstreamed recently.

Besides from the main project, we tought that it would be useful to have a profiling tool which
interfaces with perf and can be embedded directly in user code.
Also if it is intended as a tool for timing and cpu
perfomance it won't need to interface with the JIT compilation directly, as
done in [LinuxPerf.jl](https://github.com/DokFaust/LinuxPerf.jl/).

### OProfile

This was a bit tough, even if it did not required so much work.
The implementation of the OProfile wrapper can be found in LLVM, also
OProfile currently offers in the wrapper an interface to dump executed code
in a annoted file. [ref](http://oprofile.sourceforge.net/doc/devel/jit-interface.html)

As the OProfile module was working, I wanted to improve it as line annotations were not implemented.

#### DWARF debug info integration in OProfile

OProfile wrapper uses the `op_write_native_code` function to dump object
files. For line-level and code annotations it relies on
`op_write_debug_line_info` which accepts as input a pointer to the code
location, an array of `debug_line_info` entries and the number of entries.

The problem is that LLVM does not implement line-level annotations for
OProfile, which if we want to integrate that in Julia should be done
manually.

[`op_write_debug_line_info`](http://oprofile.sourceforge.net/doc/devel/op_write_debug_line_info.html) is composed of three fields : vma, line number and
filename. `vma` in the docs is defined as an [unsigned long](https://www.cs.rice.edu/~la5/doc/oprofile/d0/d27/structdebug__line__info.html#a9d5640d6fea5ff12a5545d8f8940424c) which represents the starting address of the profiling interface, which in the case of the JVM is [JVMTI](http://www.oracle.com/technetwork/articles/javase/jvmti-136367.html).

As an implementation of `op_write_debug_line_info` I have found only the
one present in the [JVM source](https://github.com/aosp-mirror/platform_external_oprofile/blob/3722f1053f4cab90c4daf61451713a2d61d79c71/agents/jvmti/libjvmti_oprofile.c) which seems a bit problematic to implement, especially for how to construct the `debug_line_info` struct using the __DWARF__ context built from JITed objects.

Finally the patch was upstreamed with commit [rL334782](https://reviews.llvm.org/rL334782) in LLVM, so it was accepted with the others patches in Julia.

### Julia interface to GDB

Julia adds a lot of useful information for user during the [debugging phase](https://docs.julialang.org/en/stable/devdocs/debuggingtips/).
In general GDB is still hooked to the Julia JIT, but before JITEventListeners notification that
was accomplished by pasting a chunk of LLVM code in `jitlayers`.
I solved that with [PR 28407](https://github.com/JuliaLang/julia/pull/28407) which cleans up
the redundant code.

### Future work

After [documenting](https://github.com/JuliaLang/julia/pull/28538) the work done, I investingated
the possibility of using `perf c2c` to trace [false sharing](https://software.intel.com/en-us/articles/avoiding-and-identifying-false-sharing-among-threads).

Even if [`perf c2c` is perfect for the purpose](https://joemario.github.io/blog/2016/09/01/c2c-blog/)
we are still incapable of using it properly with Julia.
I opened an issue on the [kernel bugtracker](https://bugzilla.kernel.org/show_bug.cgi?id=200613) but it will probably take time as it depends on how threads are traced.

## Thread safe rand

When using the `rand()` function in a multi-thread application,
 it is evident how the generated numbers are not indipendent from each other,
 this can be followed from the [issue #10441](https://github.com/JuliaLang/julia/issues/10441).

As it is presented in the discussion, this can be tackled by providing a
different initialized state for each thread, so they can evolve independently from each other.

This still comes at a cost : it is crucial to understand that we are reasoning on a
peculiar situation, when RNG requires that we have, starting from a seed, a reproducible
sequence of numbers which are uncorrelated up to a lower-bounded Shannon Entropy.
On the other hand threads are scheduled in a way that is difficult to reconstruct, so
knowing at each time which resource was consumed by whom can be unfeasible.

### The presented implementation

After a bit of time spent on rearranging the internals, due to a serious performance regression
that afterwards found explanation, we come to a solution with a thread-indexed array of
MersenneTwister generators.

The independence of generated results is guaranteed by `randjump()`, which returns an array
of generators jumped by a given polynomial (in our case is set to default, `10^20`).

The [issue](https://github.com/JuliaLang/julia/issues/27794) was that it is not possible to
call `randjump` at compile time, as it relies on `CharPoly[]`, (a string representation of the polynomial in GF2X), that is initialized at `__init()`.
This emerged only at module initialization, which can be [pretictable](https://docs.julialang.org/en/v0.6.0/manual/modules/#Module-initialization-and-precompilation-1).
This required to instatiate the `ThreadedRNG` array as a `Ref` that calls `randjump` at runtime through `TRNG()` method constructor.

To sketch it up this is the working diff:

```
diff --git a/stdlib/Random/src/RNGs.jl b/stdlib/Random/src/RNGs.jl
index 73ff51b136..80f6a71e7e 100644
--- a/stdlib/Random/src/RNGs.jl
+++ b/stdlib/Random/src/RNGs.jl
@@ -573,3 +573,13 @@ function _randjump(mt::MersenneTwister, jumppoly::DSFMT.GF2X, len::Integer)
     end
     return mts
 end
+
+const TRNG = Array{MersenneTwister}
+const ThreadRNG = Ref{TRNG}()
+
+TRNG(r::MersenneTwister=GLOBAL_RNG) = TRNG(randjump(r, big(10)^20, Threads.nthreads()))
+
+function trand(r::Array{MersenneTwister}=ThreadRNG[])
+    rt = r[Threads.threadid()]
+    return rand(rt)
+end
diff --git a/stdlib/Random/src/Random.jl b/stdlib/Random/src/Random.jl
index 4e3d01c844..27ef5c2054 100644
--- a/stdlib/Random/src/Random.jl
+++ b/stdlib/Random/src/Random.jl
@@ -248,6 +248,7 @@ function __init__()
         Base.showerror_nostdio(ex,
             "WARNING: Error during initialization of module Random")
     end
+    ThreadRNG[] = TRNG()
 end

 include("RNGs.jl")
```

During the time spent working on this, the old version of `randjump` which returned an array of
jumped generators was dismissed from the Random library. A similar method, which returned a single
jumped object, was moved to `Future`, with the intention of reintegrating it in the `Random`
codease afterwards.

My work was wrapped in the [PR #27950](https://github.com/JuliaLang/julia/pull/27950), which
included tests to check that the test case presented in the issue does not return anymore
an array with two identical entries.

The approach also included moving out of `Future` the `randjump` method and using `accumulate`
to fill the `TRNG` array.

### Alternative method : MT values as a threaded buffer

Another approach that I tried as an alternative to the threaded Array, due to the unpreticed
downgrade in performance, was to avoid generator jump at all.

To put this in practice I simply modified the routines that `rand()` uses to read and write
the MT generator. This consisted of a reading buffer that was divided between threads,
and as `idxF` reached the buffer length it called back `DSFMT` to jump forward the state.
The full implementation is available [here](https://github.com/DokFaust/julia/commit/f5c18e662ac71c442ae6791ddac7622526cc66d5).

It became clear that this approach brought more problems than it solved :
1) `idxF` is used as a buffer index, so when it reached the buffer length it triggers a call
    to the external `DSFMT` library. This of course can result in a race condition, and putting
    a lock to guard the GLOBAL_RNG would make this approach useless.
2) To overcome this, it could be possible to allow only one thread to update the idxF buffer.
This becomes problematic for a non sequential order of threads, as it would return repetitive values.

### Future work
After the 1.0 release a lot of methods have been deprecated in `Random`, which seem to bring some
troubles to possibility of merging `trand()`.

Also at the moment `trand()` is supposed to only provide a method that returns a Float64, but there
are other definitions of the method `rand()` that based on additional arguments return a different
type. It would be interesting to apply a cascade method to `trand()` that given the arguments
dispatch to the desired `rand()` equivalent.

## RCU typemap cache synchronization

### Background : why RCUs

RCU (Read Copy Update) are primitives which provide synchronization for lock-free data structures.
They become very attractive for data structures that are accessed almost through reads.

In general when we have to protect a writing operation with locks,
we introduce a considerable overhead due to the spin lock operations.

In this context, with frequent concurrent writes, it's better
to consider a safer access method that orders memory operations in sequential order,
plus avoids any access to the safe region from two concurrent writing threads.

The extra overhead is not justified instead when we have a structure that is accessed
by mostly reading threads, the time spent by constantly acquiring and releasing a spin lock
are not really justified.

That's how we come to lock-free data structures, in particular RCU where developed in the
[Linux Kernel](https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook-e1.pdf) as to coordinate memory access more easily, as the kernel already knew how threads were scheduled.

### Formal implementation of Userspace RCUs (URCU)

The idea was to introduce in userspace a similar mechanism, even if we have no real access to
the scheduler choices. We could still let threads obey a given synchronization rule by :

1) *Memory Barriers* are needed to let the hardware and compiler execution to strictly obey the
rules imposed in our program's memory model.

2) *Synchronization primitives* : here lies the whole point of our analysis, as we need to impose
on threads a sequence of clear rules to enter/exit the guarded region.

Also we did not want that the counters could saturate for any reason, provoking deadlocks when
two threads try to acquire\release the same resource. The approach was also choosen as it gave
a simpler outlook on how an entering thread have to wait for updating ones.

The approach had also the benefit of solving the [ABA problem](https://woboq.com/blog/introduction-to-lockfree-programming.html) of deleting and updating an entry from a lock-free datastructure.

In particular it was outlied as something very simiar to the implementation present in the [perf book](https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.2017.11.22a.pdf) in section `B.7` :

```
 static void rcu_read_lock(void)
 {
   __get_thread_var(rcu_reader_gp) =

   ACCESS_ONCE(rcu_gp_ctr) + 1;
   smp_mb();
 }

 static void rcu_read_unlock(void)
 {
   smp_mb();
   __get_thread_var(rcu_reader_gp) =

  ACCESS_ONCE(rcu_gp_ctr);
 }

 void synchronize_rcu(void)
 {
   int t;

   smp_mb();
   spin_lock(&rcu_gp_lock);
   ACCESS_ONCE(rcu_gp_ctr) += 2;
   smp_mb();
   for_each_thread(t) {

        while ((per_thread(rcu_reader_gp, t) & 0x1) &&
        ((per_thread(rcu_reader_gp, t) - ACCESS_ONCE(rcu_gp_ctr)) < 0)) {

            poll(NULL, 0, 10);

        }
    }
   spin_unlock(&rcu_gp_lock);
   smp_mb();
 }
```

`rcu_reader_gp` is defined as a per-thread counter, which simply signals if the corresponding thread is still in the critical zone, `rcu_gp_ctr` is a global counter.

`ACCESS_ONCE` temporarly allocates the variable as a `volatile`, which avoids any reordering
from the compiler. This is not a sufficient requirement out of kernel space, so it was substituted
with an atomic fetch that relied on `seq_cst` to keep sequential consistency.

Also another important point was the removal of memory barriers, as they were not needed and
mostly of the synchronizing work was already performed by the garbage collector.

The final pull request resembled [this](https://github.com/JuliaLang/julia/pull/28202)

### Aftermath : the consituency of a different approach

The final version of the RCu primitives almost resembled Dekker's algorithm, without the problem
of "deadlocking" flags.

Also another point of skepticism was that the protection against typemap deletion is already
provided by the garbage collector, as above.

The only remaining point where these primitives could turn out to be useful was to synchronize
reads between threads and avoid any kind of descrepancy between threads, despite deletions.
The problem is that probably those cases do not even manifest, and even if they do there is no
proper consensus that this approach could benefit much of it.

To keep the work on typemap thread safety, I first tried to wrap insertion methods in a
acquire-release mutex lock that substituted the signal fence [on PR #28512](https://github.com/JuliaLang/julia/pull/28512),
which I eventually dismissed as it emerged that during codegen method caching and generic function insertion,
the method cache is already protected by two separate locks.

So to complete the work on the data structure I added, as suggested, a bunch of atomic loads
and stores to the only cache reading method which is not included in the generating function
lock, which seems `jl_typemap_level_assoc_exact()`. The work is listed in [PR #28582](https://github.com/JuliaLang/julia/pull/28582).

## Conclusions

For sure, my favourite part of the GSoC was being involved in projects like LLVM, which just a month before staring, I would have never think of.
I found difficult, during the three months, delivering a complete implementation of each submodule which regarded different areas of the project.

In general being part of this project has been a great experience, I really had the opportunity to
show to myself what I can accomplish! The work has been great also in areas where I was not very
prepared before May, but fortunately I had been able to produce some good work.

Fortunately the areas of work have all been covered, and what was requested in the timeline
was accomplished.

During the working period, the part on RCU has been stopped, as it seemed to not being useful,
but at least something progressed to improve the safety of the typemap cache.

Anyway this had been a great time to beign part of the community, and I hope to continue on this
track in the future!
