---
title: "Graceful Shutdown Is a Feature, Not a Signal Handler"
description: "Handling SIGTERM is easy. Shutting down a Go service correctly under load is not. Graceful shutdown must be designed, not bolted on."
pubDate: 2026-01-26
tags: ["golang", "backend", "concurrency", "graceful-shutdown", "context"]
---

Graceful shutdown is often treated as an implementation detail.

Something like:
> “Catch SIGTERM, close a channel, done.”

That mindset works — until the first real production deploy under load.

Then suddenly:
- requests hang
- deploys stall
- Kubernetes kills your pod
- and nobody is quite sure why

The reason is simple:

> Graceful shutdown is not a signal handler.  
> It’s a **system-level feature**.

---

## What “graceful” actually means

A graceful shutdown guarantees that:

- no new work is accepted
- in-flight work is allowed to finish (within bounds)
- background goroutines exit
- resources are released
- the process terminates *predictably*

Anything less is wishful thinking.

---

## The common but incomplete approach

Most services start here:

```go
sig := make(chan os.Signal, 1)
signal.Notify(sig, os.Interrupt, syscall.SIGTERM)

<-sig
log.Println("shutting down")
````

This detects shutdown.

It does **not** implement shutdown.

Nothing here:

* stops goroutines
* cancels requests
* drains workers
* enforces time limits

It only *observes* the signal.

---

## Shutdown must propagate through context

In Go, cancellation is cooperative.

The only scalable way to shut down a service is:

> **cancel context → everything reacts**

A minimal pattern looks like this:

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
```

This gives you:

* a root context
* cancellation on SIGTERM
* a single authority for shutdown

Every part of the system must derive from this context.

---

## HTTP servers: stop accepting, then wait

The correct shutdown sequence for an HTTP server:

```go
go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatal(err)
    }
}()

<-ctx.Done()

shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

srv.Shutdown(shutdownCtx)
```

What this does:

* stops accepting new connections
* waits for active requests
* enforces an upper bound

Without this, requests are cut mid-flight.

---

## Goroutines must listen for cancellation

This is where many shutdowns fail.

If you have background goroutines like:

```go
go func() {
    for {
        doWork()
    }
}()
```

Your shutdown is already broken.

The correct pattern is:

```go
go func() {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            doWork()
        }
    }
}()
```

> Every long-lived goroutine must know **when to stop**.

---

## Worker pools must drain, not disappear

Worker pools need special care.

Bad shutdown:

* close the process
* workers die mid-job

Better shutdown:

* stop accepting new jobs
* let workers finish current work
* exit cleanly

Typical pattern:

```go
close(jobs)
wg.Wait()
```

But only if:

* producers are stopped
* consumers exit on channel close
* cancellation is respected

Shutdown is choreography.

---

## Time is part of correctness

A shutdown without timeouts is not graceful.

You must decide:

* how long requests may run
* how long background tasks may finish
* when to force termination

This is not pessimism.
It’s engineering.

> “Wait forever” is not a strategy.

---

## A quick Python contrast (only where it helps)

In Python async systems (e.g. FastAPI + asyncio), shutdown often relies on:

* lifespan events
* task cancellation via the event loop
* implicit propagation

Go is more explicit.

You must:

* pass context
* check cancellation
* design exit paths

This verbosity is intentional.

> Go makes shutdown behavior visible in code — not hidden in the runtime.

---

## The most common shutdown failure modes

If shutdown hangs, look for:

* goroutines blocked on channels
* goroutines ignoring context
* worker pools without drain logic
* background retries with no exit condition
* database calls without timeouts

All of these are design issues, not bugs.

---

## A simple rule that holds up

If you remember one rule:

> **Every component must know when the system is shutting down.**

If a component doesn’t:

* it will leak
* it will block
* it will be killed

Grace is intentional.

---

## The takeaway

Handling SIGTERM is trivial.

Designing a system that:

* stops accepting work
* finishes what matters
* exits predictably

…is not.

> Graceful shutdown is not something you add later.
> It’s something you design for from day one.

In Go, shutdown correctness is a feature.
Treat it like one.

---

### Related reading

* context.Context Is the Real API in Go
* Why Goroutines Leak (and How to Prove It)
* Channels Are Not Queues
* Bounded Concurrency Beats Clever Concurrency
