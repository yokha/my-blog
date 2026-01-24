---
title: "Using a Single Session Factory for Multi-Schema Databases in SQLAlchemy"
description: "How to handle multiple schemas in FastAPI with async SQLAlchemy using a single session factory — and why session-per-request is non-negotiable."
pubDate: 2026-01-25
tags: ["python", "sqlalchemy", "fastapi", "async", "database", "backend"]
---

Handling multiple schemas in a FastAPI application often sounds more complex than it actually is.

A common misconception is that each schema requires its own SQLAlchemy engine or session factory.
In reality, **a single session factory is usually enough** as long as each request gets its own session.

This post explains:
- how multi-schema setups actually work in SQLAlchemy
- two safe ways to target different schemas
- and why sharing database sessions across requests is a subtle but serious bug

---

## The core principle: sessions are request-scoped

Before talking about schemas, it’s important to get one thing right:

> **Database sessions are not reusable across requests.**

In async applications especially, a session must be:
- created per request
- used by one logical flow
- closed when that request ends

Violating this rule leads to:
- race conditions
- inconsistent reads
- broken transactions
- extremely hard-to-debug behavior

Once this principle is clear, multi-schema handling becomes much simpler.

---

## One engine, one session factory, many schemas

SQLAlchemy sessions are **not bound to a single schema**.

They are bound to:
- an engine
- a database connection (from the pool)

Schemas are resolved *at query time*, not at engine creation time.

This means:
- you don’t need multiple engines
- you don’t need multiple session factories
- you just need to control how schema resolution happens per request

---

## Approach 1: Dynamic schema switching per request

This approach uses the database’s schema resolution mechanism.
In PostgreSQL, that’s `search_path`.

### Example

```py
from sqlalchemy import text

async def get_db(schema: str):
    async with SessionLocal() as session:
        await session.execute(
            text(f"SET search_path TO {schema}")
        )
        yield session
```

Once `search_path` is set:

* all ORM queries use that schema implicitly
* models don’t need to specify a schema
* the same code works across tenants

### When this works well

* multi-tenant systems
* schema-per-customer designs
* cases where schema changes per request

### Things to be careful about

* schema values must be validated (never trust user input blindly)
* the session must not be reused
* the schema switch must happen **inside** the session lifecycle

> The schema is part of session state — not global state.

---

## Approach 2: Explicit schema in the model

When schemas are static and known ahead of time, you can bind them directly in your models.

### Example

```py
from sqlalchemy import Column, Integer, String
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    __table_args__ = {"schema": "schema1"}

    id = Column(Integer, primary_key=True)
    name = Column(String)
```

Now:

* the schema is explicit
* no runtime switching is needed
* queries are fully deterministic

### When this works well

* fixed schemas
* reporting or internal tools
* systems without per-request schema variation

The trade-off is flexibility: schema choice is now part of the model definition.

---

## Why not share a session across requests?

This is where many subtle bugs come from.

Consider what happens if you reuse a session:

* one request commits or rolls back
* another request sees unexpected state
* one request closes the session
* another request suddenly fails
* async tasks interleave unpredictably

In async environments, this can happen **without obvious errors**.

> The code may work in development and fail only under load.

This is why session-per-request is not a style preference BUT it’s a correctness requirement.

---

## The correct FastAPI pattern

FastAPI’s dependency system makes the safe pattern easy:

```py
async def get_db():
    async with SessionLocal() as session:
        yield session
```

Each request:

* gets a fresh session
* controls its own transaction
* cleans up automatically

Schema handling — whether dynamic or static — happens *inside* this boundary.

---

## Putting it all together

You usually want:

* **one engine**
* **one session factory**
* **one session per request**
* **schema resolution inside the session**

Whether you choose dynamic switching or explicit schemas depends on your data model — not on SQLAlchemy limitations.

---

## The takeaway

Handling multiple schemas in SQLAlchemy does **not** require multiple session factories.

What it does require is discipline:

* fresh sessions per request
* no shared session state
* explicit schema resolution

Get those right, and multi-schema setups stay simple, predictable, and safe — even in async FastAPI applications.
