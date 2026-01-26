---
title: "Channels Are Not Queues"
description: "Go channels look like queues, but they behave very differently. Treating them as queues leads to deadlocks, leaks, and backpressure bugs."
pubDate: 2026-01-27
tags: ["golang", "concurrency", "channels", "goroutines", "backend"]
---

If you’re coming to Go from other ecosystems, it’s easy to make a mental shortcut:

> “Channels are just queues.”

They aren’t.

They can *look* like queues.
They can *buffer* like queues.
But they behave very differently — especially under load.

And misunderstanding that difference is a common source of subtle production bugs.

---

## Why channels feel like queues at first

At a glance, a channel looks familiar:

```go
ch := make(chan Job, 10)

ch <- job
job := <-ch
````

You send.
You receive.
There’s even a buffer.

So it’s tempting to think:

* producers push
* consumers pull
* jobs wait in between

That’s the queue mental model.

But Go channels were not designed primarily as queues.

---

## What channels really are

A channel is a **synchronization primitive**.

Its primary purpose is:

* coordination between goroutines
* signaling availability
* controlling execution flow

Data transfer is secondary.

> A channel is a rendezvous point — not a storage system.

Even buffered channels preserve this property.

---

## The key difference: backpressure is immediate

Queues usually absorb pressure.

Channels **propagate pressure**.

Consider:

```go
ch := make(chan int, 1)

ch <- 1
ch <- 2 // blocks
```

Once the buffer is full:

* the sender blocks
* the system slows *at the producer*

This is intentional.

> Channels are designed to make overload visible, not hidden.

Queues often do the opposite.

---

## When treating channels like queues goes wrong

### 1. “Fire-and-forget” goroutines

```go
func submit(job Job) {
    go func() {
        ch <- job
    }()
}
```

This looks harmless.

But if:

* the channel is full
* no receiver is active

That goroutine blocks forever.

You didn’t build a queue.
You built a **goroutine leak generator**.

---

### 2. Unbounded producer, bounded consumer

```go
for _, job := range jobs {
    ch <- job
}
```

If consumers are slower:

* producers block
* request handlers stall
* timeouts cascade

Again: not a bug in Go.
A misunderstanding of intent.

---

### 3. Buffered channels as “safety nets”

Increasing buffer size feels like a fix:

```go
make(chan Job, 10000)
```

But now:

* memory grows
* latency grows
* backpressure is delayed, not removed

You’ve just hidden the problem.

---

## Channels shine when used correctly

Channels work best when:

* lifetimes are well-defined
* senders and receivers are coordinated
* cancellation is explicit
* capacity is intentional

Classic good uses:

* worker pools
* fan-in / fan-out patterns
* signaling completion
* bounded concurrency

---

## Worker pools: the right mental model

This is where channels *do* work like you expect:

```go
jobs := make(chan Job)
results := make(chan Result)

for i := 0; i < workers; i++ {
    go worker(jobs, results)
}
```

Why this works:

* workers are the limiting factor
* jobs block when workers are busy
* backpressure is controlled

The channel is not the queue.
The **workers are**.

---

## If you actually need a queue…

Be honest about it.

If you need:

* persistence
* durability
* retries
* replay
* decoupled producers and consumers

You don’t want a channel.

You want:

* a message broker
* a job queue
* a database-backed system

Examples:

* Kafka
* Redis Streams
* SQS
* a table with status + retries

> Channels are in-memory coordination tools, not infrastructure.

---

## Channels and shutdown

Another common trap:

```go
for job := range ch {
    process(job)
}
```

This loop only exits when:

* the channel is closed

If you forget to close it:

* goroutines never exit
* shutdown hangs

Channels demand **explicit lifecycle management**.

---

## A better mental model

Instead of thinking:

> “This channel stores work”

Think:

> “This channel synchronizes progress”

Ask yourself:

* who sends?
* who receives?
* who closes?
* what happens under cancellation?

If you can’t answer those clearly, the design isn’t finished.

---

## The takeaway

Channels are powerful because they are **honest**.

They don’t hide overload.
They don’t smooth spikes.
They don’t magically scale.

They force you to confront:

* capacity
* lifetimes
* cancellation
* flow control

> Channels are not queues.
> They are coordination points.

Treat them that way, and Go stays boring.
Treat them like queues, and production gets exciting.

---

### Related reading

* Why Goroutines Leak (and How to Prove It)
* context.Context Is the Real API in Go
* Bounded Concurrency Beats Clever Concurrency
* Async Doesn’t Make Your System Fast — It Makes It Honest
