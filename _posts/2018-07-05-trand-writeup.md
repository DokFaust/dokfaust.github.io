---
layout: post
title: "Implementation of a thread-safe rand"
tags: [ gsoc, julia, RNG, report ]
---

# The proposed approach

As previously presented, we needed to implement a thread-safe version of `rand()` in Julia.
The main idea is to use a thread-indexed array of MersenneTwister generators, which would allow
to keep performance as it is lock-free.

Anyway we encounter a slight performance regression (more on that later),
so I prefered to keep `rand()` autonomous for single thread programs.
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
## Performance issues among master production
Testing this solution on commit `fe4337613f0da442cf01c553c0a0a8b9dc8b66a2` (~25 days old master)
gives on the `trand()` benchmarks

```
    memory estimate:  0 bytes
    allocs estimate:  0
    --------------
    minimum time:     5.395 ns (0.00% GC)
    median time:      6.527 ns (0.00% GC)
    mean time:        6.661 ns (0.00% GC)
    maximum time:     29.691 ns (0.00% GC)
    --------------
    samples:          10000
    evals/sample:     1000
```
Instead on the more recent commit `80db579cf938a75cbb1edf0dbe90fc27d85210ac` (~2 days master)
```
    memory estimate:  16 bytes
    allocs estimate:  1
    --------------
    minimum time:     22.683 ns (0.00% GC)
    median time:      23.654 ns (0.00% GC)
    mean time:        31.057 ns (17.71% GC)
    maximum time:     38.199 μs (99.92% GC)
    --------------
    samples:          10000
    evals/sample:     996
```
I did not find any change in the optimizer that could rationalize such a downgrade in performance.
Anyway I will continue to check out how this happens.

## Alternative method : MT values as a threaded buffer

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

## Future work : RCU typemap lookup

As it remains a unsafe part of the Julia dispatch, `typemap.c` method lookup will require RCU reads
for method lookup. With respect to spin-lock reads, RCU memory barriers are faster in the order of magnitudes and allow parallel modification of the data structure.

This means that if in Thread1 the function `f()` is modified, and Thread2 is still running it, the
older `f1()` definition would still be available for Thread2 up until il reaches `synchronize_rcu`
which asynchronously waits for all threads to finish
