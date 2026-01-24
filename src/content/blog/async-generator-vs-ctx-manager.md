---
title: "The Subtle Async Mistake: Async Generators vs Async Context Managers"
description: "A real-world async bug caused by confusing a callable async generator with an async context manager — and why Python’s async protocols can quietly bite you."
pubDate: 2026-01-21
tags: ["python", "async", "fastapi", "testing", "backend", "reliability"]
---

Some async bugs don’t look like bugs.

They look like:
- broken tests
- flaky fixtures
- misconfigured dependencies
- “something weird in asyncio”

This one came from a simple assumption — and cost far more time than it should have.

---

## The mistake

I had a callable attribute that returned something I *assumed* was an async context manager.

So I wrote:

```py
async with db_session_factory() as session:
    ...
````

It looked correct.
It felt correct.
It passed type checking.
Nothing obvious was wrong.

But it wasn’t correct.

---

## What it actually was

The callable didn’t return an async context manager.

It returned an **async generator**.

That means this:

```py
db_session_factory()
```

returned something like:

```py
async def gen():
    yield value
```

Which follows the **async iteration protocol**, not the **async context manager protocol**.

So `async with` was the wrong abstraction.

---

## The correct usage

An async generator must be consumed with `async for`:

```py
async for session in db_session_factory():
    ...
```

Not:

```py
async with db_session_factory() as session:
    ...
```

Same callable.
Completely different protocol.

---

## Why this is so easy to get wrong

Because the syntax looks almost identical:

```py
async with factory()
```

vs

```py
async for item in factory()
```

But the semantics are totally different:

### Async context manager

Implements:

* `__aenter__`
* `__aexit__`

Used by:

```py
async with ...
```

### Async generator

Implements:

* `__aiter__`
* `__anext__`

Used by:

```py
async for ...
```

Python doesn’t auto-convert between these models.

---

## Why tests made it worse

The real confusion came from testing.

* Unit tests were failing
* Fixtures behaved strangely
* Mocks looked broken
* Errors appeared far from the real cause

Everything pointed to “testing issues”.

The real bug only became obvious during **integration testing**, where the runtime behavior exposed the incorrect protocol usage.

> The abstraction mismatch hid the real problem.

---

## The deeper lesson

In async Python, it’s not enough to know that something is “awaitable”.

You must know **which async protocol it implements**:

* async function -> awaitable
* async generator -> async iterable
* async context manager -> context protocol

They are not interchangeable.

---

## A simple mental model

Ask one question:

> Does this object manage a lifecycle — or produce values?

If it manages a lifecycle:

* connection
* transaction
* resource boundary
  -> context manager (`async with`)

If it produces values:

* streams
* sessions
* events
  -> async generator (`async for`)

---

## How to protect yourself from this class of bug

### 1. Make interfaces explicit

Prefer names that encode behavior:

```py
get_session()        # context manager
iter_sessions()      # async generator
```

### 2. Type your factories

```py
from typing import AsyncGenerator

def db_session_factory() -> AsyncGenerator[Session, None]:
    ...
```

This makes misuse visible earlier.

### 3. Don’t trust syntax familiarity

`async with` and `async for` look similar — but they are not interchangeable abstractions.

---

## Why this matters in FastAPI and async systems

In async services:

* resource lifecycles matter
* session management matters
* context boundaries matter
* protocol correctness matters

Mistakes don’t always explode.
They degrade behavior silently.

That’s the dangerous part.

---

## The takeaway

This wasn’t a complex bug.
It wasn’t a concurrency issue.
It wasn’t an event loop problem.

It was a **protocol mismatch**.

A callable async generator was treated like an async context manager.

> Async bugs aren’t always about race conditions.
> Sometimes they’re about abstractions that look right — but aren’t.

Understanding Python’s async protocols is part of building reliable async systems.

And sometimes, the hardest bugs come from the smallest assumptions.

---

### Related reading

- *Async Doesn’t Make Your System Fast but Honest*
- *Dynamic Scheduled Tasks in Python*
- *Using a Single Session Factory for Multi-Schema Databases in SQLAlchemy*
