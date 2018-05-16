---
layout: post
title: "Gsoc week -1 report"
tags: [ gsoc, julia, llvm, perf, report ]
---
#What my project is all about

My project on Julia Thread safety has been accepted for this GSoC. I will
focus on enhancing native profiling support, per-thread RNG and thread safe
module dispatch.

## Before the coding period

During the last month I prepered some tools that will be necessary for the
coding period. As __perf__ support is necessary to test dynamic dispatch,
I have first tried to implement a [Julia API](https://github.com/DokFaust/LinuxPerf.jl), but that seemed a bit unreliable. 
Then this [LLVM Patch](https://reviews.llvm.org/D44892) was introduced to provide native perf support in Julia, but after a first test one of my mentors, [Valentin Churavy](https://github.com/JuliaLang/julia/pull/14727#issuecomment-386840042), noticed that external profiling support was not supported.
As after the migration to LLVM Orc JIT, [JITEventlistener](https://github.com/JuliaLang/julia/issues/26999) is not implemented, so to perf, but also to tools like OProfile and VTune, JIT Symbols are not registered anymore.

## First Task : refactor JITEvents Registration

As mentioned before, Intel Vtune and OProfile were maintened by the Julia
community for external profiling support, so their maintence can be
significant for the project.
An [LLVM Review](https://reviews.llvm.org/D44892) was proposed to fix the
problem, but merging it with the current LLVM portage in Julia is
complicated by a cascade of implementation differences, mainly in RTDyld
object Linking.
This is what I have to accomplish by the end of May.

## Other activities before coding period

 * Stumbled upon Julia's internals and studied [Is Parallel Programming Hard, And, If So, What Can You Do About It?](https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html). I was mainly focused on data structures shared by different threads guarded by RCU.

 * Worked on LLVM tutorials and [pass writing](https://cseweb.ucsd.edu/classes/sp14/cse231-a/proj1.html).

## Other projects for the summer

1) About thread-safe RNG, I discussed with my mentors of the most effective way to tackle it and we agreed on a thread-indexed vector, as in [KissThreading.jl](https://github.com/bkamins/KissThreading.jl/blob/master/src/KissThreading.jl#L7-L13).

1) To test module dispatch I will write a script to test multiple function declaration - override among different threads. For this task [FunctionWrappers](https://github.com/yuyichao/FunctionWrappers.jl) will be useful.
