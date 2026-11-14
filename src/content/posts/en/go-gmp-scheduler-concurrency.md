---
title: "Goroutines vs. Threads vs. Processes and the Go GMP Scheduler"
author: "John Yang"
lang: en
pubDatetime: 2023-06-01T05:50:00Z
tags:
  - Go
  - Concurrency
  - Architecture
description: "Comparing processes, OS threads, and goroutines, followed by a deep dive into Go's GMP scheduler, work stealing, handoff, and lifecycle."
---

# Core Concepts

- **Process**: Each process has its own isolated memory space, including its own address space, heap, and stack (neither heap nor stack is shared across processes). A program has at least one process, and a process has at least one thread. Process context switches happen exclusively in kernel space.
- **Thread**: A thread has its own stack while sharing the process heap with other threads in the same process. Threads are scheduled preemptively by the OS kernel using CPU time slices, which incurs kernel-level context-switching overhead.
- **Coroutine / Goroutine**: Like threads, coroutines share the heap and keep separate stacks, but they are scheduled in user space rather than by the OS kernel—meaning the kernel is completely unaware of them. A goroutine starts with a tiny stack (~2 KB, compared to ~8 MB default thread stacks on Linux) that grows and shrinks dynamically as needed.
- **Concurrency**: Multiple tasks making progress by interleaving on a single CPU core.
- **Parallelism**: Multiple tasks executing simultaneously across multiple CPU cores.

**The key difference between a goroutine and an OS thread is who schedules it:** OS threads are scheduled by the kernel, whereas goroutines are scheduled in user space by Go's runtime scheduler.

***

# Deep Dive

## Processes

On a single-CPU system, a processor can execute only one instruction stream at a time. Letting a single program monopolize the CPU—especially while blocked waiting on network I/O—wastes hardware resources. Multiprocessing solves this by letting the OS kernel switch between independent processes so multiple applications can run concurrently.

## Threads

If multiprocessing already exists, why do we need threads?

1. **Data sharing**: Processes do not share memory; exchanging data requires Inter-Process Communication (IPC), which adds overhead.
2. **Creation cost**: Spawning a new process (e.g., via `fork`) is relatively expensive.

Multiple threads inside the same process share code and global heap data, eliminating IPC overhead for shared state and making thread creation and switching significantly cheaper than processes.

## Goroutines

From the official [Go FAQ on goroutines](https://go.dev/doc/faq#goroutines):

> Goroutines are part of making concurrency easy to use. The idea, which has been around for a while, is to multiplex independently executing functions—coroutines—onto a set of threads. When a coroutine blocks, such as by calling a blocking system call, the run-time automatically moves other coroutines on the same operating system thread to a different, runnable thread so they won't be blocked. The programmer doesn't see any of this, which is the point. The result, which we call goroutines, can be very cheap: they have little overhead beyond the memory for the stack, which is just a few kilobytes.
>
> To make the stacks small, Go's run-time uses resizable, bounded stacks. A newly minted goroutine is given a few kilobytes, which is almost always enough. When it isn't, the run-time grows (and shrinks) the memory for storing the stack automatically, allowing many goroutines to live in a modest amount of memory. The CPU overhead averages about three cheap instructions per function call. It is practical to create hundreds of thousands of goroutines in the same address space. If goroutines were just threads, system resources would run out at a much smaller number.

Because goroutines are user-space lightweight execution units multiplexed onto OS threads, they offer four major advantages:

- **Lower CPU overhead**: Avoids frequent kernel-level thread context switches.
- **Lower memory footprint**: On 64-bit Linux, an OS thread reserves ~8 MB of stack space, limiting thread count. Goroutines start with only a few kilobytes, making hundreds of thousands of concurrent goroutines practical.
- **Developer productivity**: Blocking I/O (files, network requests) can be written in straightforward synchronous-looking code while the runtime handles asynchronous multiplexing under the hood.
- **Fast switching**: Switching goroutines happens in user space without timer interrupts or kernel syscalls.

***

# The Go GMP Scheduler

> **Note:** The GMP scheduler breakdown below is adapted from AceLd's (Danbing Liu) article: [[Golang Trilogy] Complete Analysis of Go GMP Scheduler Principles and Scheduling](https://learnku.com/articles/41728).

## Overview

- **`G` (Goroutine)**: Created by each `go` statement; holds its own stack, instruction pointer, and scheduling state.
- **`M` (Machine / OS Thread)**: Kernel-level thread. A `G` must be bound to an `M` to execute.
- **`P` (Processor)**: Logical scheduling context that binds `G`s to `M`s and maintains a local run queue of runnable `G`s.

The Go scheduler (`P`) bridges to the OS scheduler through `M`: each `M` represents one OS thread, and the OS scheduler assigns `M`s to physical CPU cores.

## Thread-to-Coroutine Mapping Models

1. **`N:1` Model**: `N` coroutines mapped onto `1` OS thread. Fast user-space switching, but cannot utilize multiple CPU cores—and if one coroutine blocks on a syscall, the entire process stalls.
2. **`1:1` Model**: `1` coroutine mapped to `1` OS thread. Easy to implement and utilizes multiple cores, but creation, deletion, and context switching are all paid at OS-thread cost.
3. **`M:N` Model**: `M` coroutines multiplexed onto `N` OS threads. Combines the strengths of both while avoiding their bottlenecks, at the cost of a more complex scheduler. Unlike preemptive OS threads, classic coroutines are cooperative; Go's runtime adds preemption to keep goroutines fair.

## Evolution of the Go Scheduler

Go's current scheduler was redesigned in **2012**.

### Pre-2012 Scheduler

Every `M` pulled and pushed `G`s from a single **global `G` queue** protected by a mutex:

- Heavy **lock contention** on every create, destroy, and schedule operation.
- Poor **cache locality** when `M`s handed off newly spawned child `G`s through the global queue.
- Frequent **syscall blocking/unblocking** overhead across `M`s.

### Post-2012 GMP Scheduler

The 2012 redesign introduced **`P` (Processor)** between `M` and `G`, giving each `P` its own local run queue:

1. **Global Queue**: Fallback queue holding runnable `G`s.
2. **Local Queue of `P`**: Holds up to **256** runnable `G`s. Newly created `G`s go into the current `P`'s local queue first; if full, half of the local queue is moved to the global queue.
3. **`P` List**: Created at startup in an array of size `GOMAXPROCS` (configurable).
4. **`M`**: An OS thread must acquire a `P` to run `G`s. When `P`'s local queue is empty, `M` checks the global queue or **steals half** of the `G`s from another `P`'s local queue.

## Scheduler Design Strategies

- **Thread Reuse**:
  1. **Work Stealing**: When an `M`'s bound `P` has no runnable `G`s, it steals `G`s from another `P` instead of parking or destroying the thread.
  2. **Handoff**: When a running `G` blocks in a syscall, `M` detaches from `P` and hands `P` off to another idle (or newly created) `M` so other `G`s on `P` continue executing.
- **Parallelism**: `GOMAXPROCS` controls the number of `P`s—limiting how many OS threads execute Go code simultaneously across CPU cores.
- **Preemption**: Unlike pure cooperative coroutines, Go limits a goroutine's continuous run slice (~10 ms) to prevent starvation.
- **Global Queue Fallback**: Still exists as a safety net when work stealing finds no local work.

## `go func()` Scheduling Flow

1. `go func()` creates a new `G`.
2. The new `G` is placed into the current `P`'s local queue (or the global queue if the local queue is full).
3. An `M` must hold a `P` (`1:1` pairing while running Go code) and pops a runnable `G` from `P`'s local queue; if empty, it steals from other `P`s or pulls from the global queue.
4. `M` runs `G` in a scheduling loop.
5. If `G` executes a blocking syscall, `M` detaches from `P` so another `M` can take `P` and keep running other `G`s.
6. When the syscall returns, the `G` tries to acquire an idle `P`; if none is available, `M` goes to sleep in the idle thread pool and `G` is placed in the global queue.

## Scheduler Lifecycle (`M0` and `G0`)

- **`M0`**: The initial main OS thread (stored in global `runtime.m0`, not heap-allocated), responsible for runtime initialization and launching the first `G`. Afterward, `M0` behaves like any other `M`.
- **`G0`**: The special scheduling goroutine created for each `M`. `G0` does not run user code; its stack is used exclusively by the runtime scheduler and syscall transitions.

Tracing a minimal program:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello world")
}
```

1. The runtime initializes `m0` and `g0` and binds them together.
2. It initializes `m0`, stacks, the garbage collector, and creates `GOMAXPROCS` `P`s.
3. The runtime creates the `main goroutine` for `runtime.main` (which invokes `main.main`) and enqueues it on `P`'s local queue.
4. `m0` (bound to a `P`) pops the `main goroutine` from the local queue.
5. `M` sets up the execution context from `G`'s stack and scheduling state and runs `G`.
6. When `main.main` exits, `runtime.main` handles `defer`/`panic` cleanup and calls `runtime.exit`.

The scheduler's lifecycle spans the entire lifetime of a Go program—initializing before `runtime.main` starts and terminating only when `runtime.main` exits.

***
