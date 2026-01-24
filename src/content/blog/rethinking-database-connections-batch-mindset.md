---
title: "Rethinking Database Connections: Engine, Session, and the Batch Mindset"
description: "Most database performance problems don’t start in the query. They start in how we connect, transact, batch, and recover under load."
pubDate: 2026-01-24
tags: ["backend", "databases", "sqlalchemy", "reliability", "performance"]
---

Most database performance issues don’t start in the query.

They start **before** that — in how we connect, how we scope sessions, how we commit,
and how we react when something goes wrong under load.

Over time, I’ve found it useful to think about database interaction in **three layers**:

- the **engine** (connections)
- the **session** (consistency and isolation)
- the **batch mindset** (how work is grouped, retried, and recovered)

Miss one of these, and systems that look fine in development start to stall or behave unpredictably in production.

---

## Engine: the lifecycle of connections

The engine defines **how your application talks to the database**.

It controls:
- connection pooling
- max concurrency
- connection reuse
- how fast you hit limits under load

A misconfigured engine doesn’t fail loudly.
It just slows everything down.

Typical symptoms:
- requests pile up waiting for connections
- timeouts appear “randomly”
- increasing workers makes things worse, not better

> The engine is not an implementation detail.  
> It is a capacity decision.

---

## Session: isolation, not convenience

A session is not “a connection with helpers”.

A session represents:
- a unit of work
- a consistency boundary
- a transactional context

Two important rules that keep coming back:

- **Sessions are not reusable across requests**
- **Sessions should be short-lived and explicit**

Sharing a session across requests can lead to:
- stale reads
- accidental cross-request commits
- subtle race conditions in async environments

> A clean session lifecycle is the foundation of predictable behavior.

---

## Where things really break: one-operation-at-a-time thinking

Even with a good engine and clean sessions, many systems fail at the next level.

They treat **every operation as isolated**.

```

BEGIN
do one thing
COMMIT

BEGIN
do one thing
COMMIT

BEGIN
do one thing
COMMIT

```

This works… until:
- traffic spikes
- locks collide
- retries multiply
- failures cascade

Every transaction becomes:
- a new failure surface
- a new retry storm
- a new chance to deadlock

The missing idea is **batching with intention**.

---

## The batch mindset

>Batching isn’t about speed. It’s about **failure boundaries**.

A good batch strategy lets you:
- group related work
- retry only what failed
- keep partial success when appropriate
- avoid poisoning the entire transaction

Instead of thinking:
> “How fast can I execute this operation?”

Think:
> “What happens when the third item fails?”

---

## SAVEPOINTs: controlled failure inside a transaction

Relational databases give us a powerful tool for this: **SAVEPOINTs**.

They let you:
- keep one outer transaction
- isolate failures per item
- retry or skip without losing all progress

### Mental model

```

BEGIN TRANSACTION
for each item:
SAVEPOINT item
try operation
if transient failure:
ROLLBACK TO SAVEPOINT
retry
if permanent failure:
ROLLBACK TO SAVEPOINT
record error
COMMIT

````

This is how you stay predictable under pressure.

---

## Async SQLAlchemy example: safe batch updates

Here’s a concrete example using async SQLAlchemy.

One transaction.
One SAVEPOINT per item.
No global rollback unless you choose it.

```py
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession


async def apply_batch(session: AsyncSession, updates):
    results = {"ok": [], "failed": []}

    async with session.begin():  # outer transaction
        for user_id, new_email in updates:
            try:
                async with session.begin_nested():  # SAVEPOINT
                    await session.execute(
                        text("UPDATE users SET email = :email WHERE id = :id"),
                        {"email": new_email, "id": user_id},
                    )
                results["ok"].append(user_id)

            except Exception as exc:
                # rolled back to SAVEPOINT automatically
                results["failed"].append((user_id, str(exc)))

    return results
````

Key properties:

* failures don’t poison the whole batch
* retries can be scoped per item
* commits stay intentional

---

## Timeouts and retries belong at the operation level

Another common mistake:

* setting timeouts only on the engine
* retrying entire requests blindly

Instead:

* set **per-operation** timeouts
* retry only **transient failures**
* keep retries **inside** the batch logic

In PostgreSQL, for example:

```sql
SET LOCAL lock_timeout = '1500ms';
SET LOCAL statement_timeout = '3000ms';
```

> If an operation hangs, it should fail fast and recover — not stall a worker.

---

## Why I built dbop-core

After reimplementing this pattern across multiple services, I extracted it into a small library:
**dbop-core**.

The goal wasn’t abstraction for its own sake.
It was to make these principles *hard to forget*.

What dbop-core focuses on:

* automatic retries with backoff for transient errors
* per-operation timeouts
* SAVEPOINT-based safety for batch operations
* explicit failure classification
* optional OpenTelemetry instrumentation

It’s not about avoiding deadlocks at all costs.

It’s about building systems that:

* degrade gracefully
* recover predictably
* remain observable under load

You can find it here:
>👉 [https://github.com/yokha/dbop-core](https://github.com/yokha/dbop-core)

---

## The real takeaway

Performance problems rarely come from “slow queries”.

They come from:

* unclear connection lifecycles
* sloppy session boundaries
* unbounded retries
* and treating every operation as isolated

Think in layers:

* **Engine** defines capacity
* **Session** defines consistency
* **Batching** defines resilience

Once you get those right, the database stops being a mystery
and starts behaving like a system you can reason about.

That’s when production debugging gets easier.

---

### Related reading

* Async Doesn’t Make Your System Fast — It Makes It Honest
* Python Async Made Simple: Process vs Thread vs Coroutine
* Safe Dynamic Scheduled Tasks in Python
* dbop-core: predictable retries and batch execution
  [https://github.com/yokha/dbop-core](https://github.com/yokha/dbop-core)
---