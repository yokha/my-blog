---
title: "Python Async Made Simple: Process vs Thread vs Coroutine"
description: "A practical mental model for Python concurrency: processes, threads, and async coroutines—what they are, when to use them, and why FastAPI can surprise you."
pubDate: 2026-01-24
tags: ["python", "async", "concurrency", "fastapi", "backend"]
---

Concurrency in Python is often explained in pieces: processes here, threads there, async somewhere else.

What’s missing is a **single mental model** that explains how they relate and especially if you run Python web services.

This article is that model.

No framework wars, no deep internals. Just enough clarity to stop making accidental mistakes.


---

## The three execution models (one sentence each)

- **Process**: an isolated Python interpreter with its own memory.
- **Thread**: multiple execution paths sharing the same memory inside one process.
- **Coroutine (async/await)**: cooperative tasks running inside an event loop, usually on a single thread.

> They solve different problems. Mixing them without understanding the boundaries is where bugs appear.

---

## Start with the big picture

### Processes

```
[Master Process]
├─ Worker Process
├─ Worker Process
└─ Worker Process
```

- Each worker has its **own interpreter and memory**
- No shared globals between workers
- Strong isolation, higher overhead

> This is why process-based servers are robust by default.

---

### Threads (inside one process)

```
[Worker Process]
├─ Thread A
├─ Thread B
└─ Thread C
```

- Threads **share memory**
- Faster communication
- Shared state must be handled carefully

> In CPython, threads are constrained by the GIL, but they are still useful for I/O.

---

### Coroutines (async) inside one thread

```
[Worker Process]
└─ Event Loop (single thread)
├─ coroutine A
├─ coroutine B
└─ coroutine C
```

- One thread
- Many tasks in flight
- Tasks **switch** only when they **`await`**

> This is the model used by async frameworks like FastAPI.

---

## What “async” actually means

Async does **not** mean parallel execution.

Async means:
- work does not block while waiting on I/O
- tasks voluntarily yield control at `await`
- progress is driven by an event loop

> Async is about **waiting efficiently**, not doing more work at once.

> This distinction matters.

---

## Process: isolation by design

A process gives you a clean boundary:
- its own heap
- its own globals
- its own failure domain

> That’s why Gunicorn and similar servers use processes by default.

Processes are a good fit when:
- you want CPU parallelism
- isolation matters more than memory usage
- crashing one worker should not affect others

> The trade-off is cost: processes are heavier, and sharing data requires explicit mechanisms (IPC, Redis, queues).

---

## Thread: shared memory, shared responsibility

Threads live inside a process and share everything:
- globals
- objects
- caches

In CPython, the GIL means only one thread executes Python bytecode at a time, but threads are still useful when:
- waiting on blocking I/O
- integrating legacy synchronous libraries
- running background work alongside request handling

> The danger is subtle:
> shared state + concurrency requires discipline.

---

## Coroutine: cooperative concurrency

Coroutines are different.

They don’t run in parallel.
They **interleave**.

> Only one coroutine runs at a time on a thread, but execution switches whenever a coroutine awaits.

> This makes async efficient for I/O-heavy workloads — but it also makes certain bugs easier to introduce.

---

## The FastAPI + Gunicorn reality

A common production setup looks like this:

```
Gunicorn Master
├─ Worker Process
│    └─ Event Loop Thread
│         ├─ request coroutine
│         ├─ request coroutine
│         └─ request coroutine
├─ Worker Process
│    └─ Event Loop Thread
└─ Worker Process
     └─ Event Loop Thread
````

Two important consequences fall out of this.

---

### 1. Many requests share one thread

Inside a worker, **all async requests run on the same thread**.

That means:
- thread-local assumptions don’t apply the way you expect
- blocking code blocks *everything* in that worker
- CPU-heavy loops inside `async def` are dangerous

> Async makes concurrency visible, not free.

---

### 2. Globals fail in quieter ways

Globals behave differently in async systems.

Consider this pattern:

```py
current_user = None

async def handler():
    global current_user
    current_user = "alice"
    await some_io()
    return current_user
````

What can go wrong?

Between `await some_io()` and the return:

* another request may run
* it may overwrite `current_user`

No exception.
No warning.
Just wrong behavior under load.

> This is why async bugs often appear *after* deployment.

---

## How to think safely about state

In async systems:

* prefer passing state explicitly
* use request-scoped context (dependencies, context variables)
* push shared state to external systems (Redis, databases)
* treat globals as read-only unless proven otherwise

> Async rewards explicit design.

---

## When to use what (simple rules)

* **Processes**: CPU-bound work, isolation, safety
* **Threads**: blocking I/O, integration with sync libraries
* **Async coroutines**: high-concurrency I/O, controlled resource usage

> Most real systems use **all three** but in different layers.

---

## The takeaway mental model

If you remember only one thing, remember this:

- **Processes isolate.**
- **Threads share.**
- **Coroutines interleave.**

> Async doesn’t make your system faster by default.

> It makes your system **honest** about how it actually behaves under load.

---
