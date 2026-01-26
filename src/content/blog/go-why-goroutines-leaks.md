---
title: "Why Goroutines Leak (and How to Prove It)"
description: "Goroutine leaks don’t crash your Go service. They slowly poison it. Here’s why they happen, how to spot them, and how to prevent them."
pubDate: 2026-01-27
tags: ["golang", "concurrency", "goroutines", "context", "backend"]
---

Goroutines are one of Go’s greatest strengths.

They’re cheap.
They’re easy to spawn.
They scale beautifully.

And that’s exactly why they’re dangerous.

> Goroutine leaks don’t fail loudly.  
> They fail slowly — and usually in production.

---

## What a goroutine leak really is

A goroutine leak happens when a goroutine:

- is started
- never completes
- never gets cancelled
- and nobody is waiting for it anymore

The program keeps running.
Memory usage creeps up.
Latency degrades.
Shutdowns hang.

No panic.
No stack trace.
Just a system that gets… tired.

---

## The classic leak pattern

Here’s the most common shape:

```go
func handleRequest() {
    ch := make(chan Result)

    go func() {
        res := doWork()
        ch <- res
    }()

    return
}
````

What happens?

* `handleRequest` returns
* nobody is receiving from `ch`
* the goroutine blocks forever on `ch <- res`

That goroutine is now leaked.

> If a goroutine can block, it must have a cancellation path.

---

## Channels don’t magically clean up goroutines

A common misconception:

> “When the function returns, the goroutine will stop.”

It won’t.

Goroutines live independently of stack frames.
Once started, they only stop if:

* they return
* or the process exits

Nothing else kills them.

---

## Context is the missing escape hatch

The correct version uses context:

```go
func handleRequest(ctx context.Context) {
    ch := make(chan Result)

    go func() {
        select {
        case ch <- doWork():
        case <-ctx.Done():
            return
        }
    }()
}
```

Now the goroutine has a way out.

If the request is cancelled:

* `ctx.Done()` fires
* the goroutine exits
* no leak

> Goroutines must be *cooperatively cancellable*.

---

## Leaks hide behind “successful” code

This is why goroutine leaks are tricky.

Your code can:

* pass tests
* work locally
* handle light traffic

And still leak.

The failure only appears when:

* load increases
* timeouts occur
* clients disconnect
* retries pile up

> Leaks grow with traffic, not with bugs.

---

## Another common leak: fan-out without fan-in

```go
for _, id := range ids {
    go process(id)
}
```

What happens if:

* `process` blocks?
* the caller times out?
* shutdown begins?

Nothing stops these goroutines.

Correct pattern:

```go
for _, id := range ids {
    go func(id int) {
        select {
        case <-ctx.Done():
            return
        default:
            process(id)
        }
    }(id)
}
```

Or better: use a worker pool with bounded concurrency.

---

## How to *prove* you have a leak

This is the part many people skip.

### 1. Check goroutine count

In production or tests:

```go
runtime.NumGoroutine()
```

If this number:

* only goes up
* never stabilizes

You’re leaking.

---

### 2. Dump goroutine stacks

Send `SIGQUIT` or use `pprof`:

```bash
kill -QUIT <pid>
```

or:

```go
net/http/pprof
```

Look for:

* goroutines blocked on channels
* goroutines stuck on receives
* goroutines waiting on timers

Leaks leave fingerprints.

---

### 3. Write cancellation tests

The best prevention is testing behavior under cancellation:

```go
ctx, cancel := context.WithCancel(context.Background())
cancel()

doWork(ctx)

// assert goroutines exit
```

If tests hang or counts increase, you found the leak early.

---

## Goroutines and shutdown

Leaked goroutines are why services:

* refuse to shut down
* ignore SIGTERM
* hang forever during deploys

If shutdown doesn’t propagate context:

* goroutines keep waiting
* resources never release
* deploys stall

> Graceful shutdown is just structured cancellation.

---

## A simple rule that saves systems

If you remember only one rule:

> **If you start a goroutine, you must know how it stops.**

Not “probably stops”.
Not “should stop”.

How. Exactly.

---

## The takeaway

Goroutines are cheap — but not free.

They:

* don’t auto-cancel
* don’t auto-clean
* don’t respect timeouts unless you tell them to

Go gives you the tools:

* context
* channels
* select
* cancellation

But it assumes discipline.

> Goroutine leaks are not a Go problem.
> They’re a design problem.

---

### Related reading

* context.Context Is the Real API in Go
* Async Doesn’t Make Your System Fast — It Makes It Honest
* Graceful Shutdown Is a Feature, Not a Signal Handler
* Channels Are Not Queues
