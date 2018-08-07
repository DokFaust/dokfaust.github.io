---
layout: post
title: "Userspace RCU in Julia : an experiment"
tags: [ gsoc, julia, RCU ]
---
## Background : why RCUs

RCU (Read Copy Update) are primitives which provide synchronization for lock-free data structures.
They become very attractive for datastructures that are accessed almost through reads.

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

## Formal implementation of Userspace RCUs (URCU)

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

## Aftermath : the consituency of a different approach

The final version of the RCu primitives almost resembled Dekker's algorithm, without the problem
of "deadlocking" flags.

Also another point of skepticism was that the protection against typemap deletion is already
provided by the garbage collector, as above.

The only remaining point where these primitives could turn out to be useful was to synchronize
reads between threads and avoid any kind of descrepancy between threads, despite deletions.
The problem is that probably those cases do not even manifest, and even if they do there is no
proper consensus that this approach could benefit much of it.

As for the future, it would be more interesting to find a different approah, which in order
to provide a safe and lock-free typemap would mostly rely on atomic operations, which could
also provide a proper guard against concurrent write-read threads.
