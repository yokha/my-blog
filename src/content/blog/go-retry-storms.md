---
title: "Retry Storms: The Silent System Killer"
description: "Retries feel safe — until they amplify failure instead of absorbing it. How retry storms form, why they’re dangerous, and how to design retries that don’t take your system down."
pubDate: 2026-01-27
tags: ["golang", "backend", "resilience", "retries", "distributed-systems"]
---

Retries are one of those things that *feel* obviously correct.

Something fails?
Just retry.

Most of us have written code like this without thinking twice:

```go
for i := 0; i < 3; i++ {
    err := call()
    if err == nil {
        return nil
    }
}
````

It works.
It feels defensive.
It ships.

And then one day, under real load, your system collapses — not because something failed, but because **everything retried at once**.

That’s a retry storm.

---

## What is a retry storm?

A retry storm happens when:

* many requests fail around the same time
* each failure triggers retries
* retries hit the same degraded dependency
* load increases instead of decreases
* recovery becomes impossible

Nothing is “wrong” with retries individually.

The failure emerges **collectively**.

> Retry storms don’t cause outages.
> They prevent recovery.

---

## The subtle trap: retries amplify load

Imagine this sequence:

1. A downstream service slows down
2. Requests start timing out
3. Callers retry immediately
4. Downstream gets *more* traffic
5. Latency increases further
6. More retries happen

You’ve now built a positive feedback loop.

The system isn’t broken.
It’s overwhelmed.

---

## Why retry storms are so dangerous

Retry storms are hard to diagnose because:

* logs show “handled errors”
* metrics show high throughput
* no single component looks obviously broken
* everything is technically “working”

But latency keeps climbing.
Error rates stay high.
Pods restart.
Connections churn.

> The system is busy failing.

---

## Common retry storm patterns

### 1. Immediate retries

```go
for {
    if err := call(); err == nil {
        return
    }
}
```

This is the worst case:

* no delay
* no limit
* no cancellation

A single failure can melt your system.

---

### 2. Synchronized retries

Even with limits:

```go
time.Sleep(1 * time.Second)
```

If thousands of callers retry with the same delay:

* they wake up together
* they hit the dependency together
* they fail together

This is called the **thundering herd** problem.

---

### 3. Retries without deadlines

If retries don’t respect context:

* they ignore timeouts
* they ignore shutdown
* they pile up during deploys

Retries must be *cancellable*.

---

## Why retries feel safe (but aren’t)

Retries give psychological comfort:

* “We handled failure”
* “We didn’t give up”
* “The system is resilient”

But resilience is not about retrying harder.

> Resilience is about knowing when to stop.

---

## Designing retries that don’t kill you

### 1. Always bound retries

Retries without limits are not retries.
They’re loops.

```go
for i := 0; i < maxRetries; i++ {
    ...
}
```

Bound everything:

* retry count
* retry duration
* total time spent

---

### 2. Add jitter — always

Never retry on a fixed schedule.

```go
sleep := base + rand.Jitter()
time.Sleep(sleep)
```

Jitter breaks synchronization.
It gives systems room to breathe.

> No jitter = herd behavior.

---

### 3. Respect context

Retries must stop when the work is no longer valid.

```go
select {
case <-time.After(delay):
    retry()
case <-ctx.Done():
    return ctx.Err()
}
```

If the request is cancelled, retries should stop immediately.

---

### 4. Separate *retryable* from *non-retryable* errors

Not every error deserves a retry.

Retry:

* timeouts
* transient network failures
* temporary overloads

Do not retry:

* validation errors
* authorization failures
* business rule violations

> Retrying permanent errors just wastes capacity.

---

### 5. Combine retries with backpressure

Retries must compete for the same resources as normal work.

If your system is overloaded:

* retries should slow down
* or be dropped

Retries are *not* special.

---

## Circuit breakers are not optional

At some point, retries must give up.

Circuit breakers:

* stop sending traffic to a failing dependency
* allow recovery time
* protect the rest of the system

Retries without circuit breakers just prolong failure.

---

## A quick Python comparison (only to clarify)

In async Python systems:

* retries often live in middleware
* tasks can silently pile up in the event loop
* cancellation is easy to forget

Different runtime.
Same problem.

> Retry storms are a design issue, not a language issue.

---

## How retry storms show up in production

If you see:

* high CPU with low useful throughput
* databases at max connections
* repeated timeouts with no clear root cause
* services that *never quite recover*

Suspect retries.

Especially “helpful” ones.

---

## A simple mental model

Retries should behave like:

* **pressure valves**, not pumps
* **shock absorbers**, not amplifiers

If a dependency is failing:

* reduce pressure
* spread retries
* stop when necessary

---

## The takeaway

Retries are powerful.
They’re also dangerous.

> The goal of retries is not to succeed at all costs.
> It’s to give the system a chance to recover.

Bound them.
Jitter them.
Cancel them.
And know when to stop.

Because the most dangerous failures are not crashes.

They’re systems that refuse to let themselves heal.

---

### Related reading

* Bounded Concurrency Beats Clever Concurrency
* Graceful Shutdown Is a Feature, Not a Signal Handler
* Channels Are Not Queues
* Async Doesn’t Make Your System Fast — It Makes It Honest
