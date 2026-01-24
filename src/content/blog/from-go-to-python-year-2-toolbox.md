---
title: "From Go to Python: What’s in Your Year-2 Backend Toolbox?"
description: "After a year building production systems with FastAPI, I’m shifting focus from shipping features to observability, auditability, and long-term correctness. Here’s how my backend toolbox is evolving."
pubDate: 2026-01-21
tags: ["python", "fastapi", "backend", "observability", "testing", "architecture"]
---

After a full year building production services with FastAPI — coming from a Go background — my focus has changed.

Year one was about **shipping**:
- learning the ecosystem
- getting things into production
- understanding async, dependencies, and performance trade-offs

Year two is about something else entirely:

> building services that are observable, auditable, testable, and calm under pressure.

---

## What Go taught me early

In Go, certain habits come almost by default.

A typical backend toolbox looked like:
- structured logging (`zap`, `logrus`)
- metrics and alerting (Prometheus, Datadog)
- distributed tracing (OpenTelemetry)
- clear interfaces and explicit contracts
- clean test boundaries
- BDD or contract testing when behavior mattered

These weren’t “nice to have”.
They were part of how you *reasoned* about the system.

When something broke, you had:
- logs that told you *what*
- metrics that told you *how bad*
- traces that told you *where*

---

## Python felt different — at first

Python makes it very easy to:
- move fast
- write expressive code
- glue systems together

FastAPI amplifies this:
- async by default
- great ergonomics
- automatic OpenAPI
- excellent developer experience

In year one, that was perfect.

But over time, familiar questions came back:

- How do I **trust** this system under load?
- How do I **audit** changes after the fact?
- How do I **trace** behavior across services?
- How do I test async behavior without lying to myself?

That’s where year two begins.

---

## The questions that now matter to me

When building production-grade FastAPI services, I now think in terms of *capabilities*, not frameworks.

### 1. Structured logging

Not:
> “Can I log something?”

But:
> “Can I reconstruct what happened across requests and services?”

Questions I care about:
- are logs structured or free-text?
- can I attach request IDs, user IDs, job IDs?
- are logs async-safe?

Tools I’ve explored:
- `structlog`
- standard `logging` with discipline
- context-aware logging patterns

---

### 2. Auditing and change tracking

Not every system needs auditing.

But when it does, it needs to be **boring and reliable**.

I care about:
- who changed what
- when it changed
- why it changed (if available)

Patterns and tools I’ve looked at:
- database-level audit tables
- SQLAlchemy versioning approaches
- `sqlalchemy-continuum`-style patterns
- event-based audit logs

> Auditing is not about features — it’s about trust.

---

### 3. Distributed tracing

Async systems hide latency very well.

Tracing brings it back into the light.

What I want:
- request-level traces
- DB and external call spans
- correlation between services

The obvious answer here is:
- OpenTelemetry

But the real work is not installing it — it’s deciding:
- what deserves a span
- what context must propagate
- what *not* to trace

---

### 4. Async-safe testing

Async changes testing semantics.

Things that “work” can still be wrong.

I care about:
- deterministic async tests
- isolation between test cases
- no hidden event-loop leaks

Tools and patterns:
- `pytest`
- `pytest-asyncio`
- careful fixture scoping
- integration tests that reveal scheduling bugs

> Async bugs rarely fail loudly.  
> Tests must be designed to catch silence.

---

### 5. Behavior and contract testing

Unit tests are not enough for backend systems.

I care about:
- API contracts
- event schemas
- behavior under failure

This doesn’t have to mean heavy BDD frameworks, but it does mean:
- explicit expectations
- stable contracts
- tests that reflect real usage

---

### 6. Monitoring and APM

Metrics are still the fastest signal.

For many systems:
- Prometheus + Grafana is enough
- simple alerts beat clever dashboards

As systems grow:
- logs explain *why*
- traces explain *where*
- metrics explain *how much*

The trick is choosing the **right level of complexity** — not the most fashionable stack.

---

## The toolbox mindset

The biggest shift from year one to year two is this:

> Tools are not about productivity — they’re about confidence.

I’m no longer asking:
> “What’s the fastest way to build this?”

I’m asking:
> “Can I explain, debug, and evolve this system six months from now?”

---

## An open question

I’ve explored tools like:
- `structlog`
- `pytest-asyncio`
- `sqlalchemy-continuum`
- `opentelemetry`

But toolboxes are shaped by experience, not blog posts.

So I’m curious:

> If you’re building production FastAPI services today,  
> what’s in *your* year-2 toolbox?

Not theory.
Not demos.
Real systems.

---

### Related reading

- Async Doesn’t Make Your System Fast — It Makes It Honest
- Python Async Made Simple: Process vs Thread vs Coroutine
- FastAPI Swagger: Auto-Magic… Until You Need Control
- Rethinking Database Connections: Engine, Session, and Batching
