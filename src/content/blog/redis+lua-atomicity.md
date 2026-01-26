---
title: "Redis + Lua: Atomic Operations Without Lying to Yourself"
description: "Why Redis Lua scripts are more than an optimization trick — and how they help you build atomic, predictable operations without race conditions or round-trip chaos."
pubDate: 2026-01-27
tags: ["redis", "lua", "atomicity", "backend", "performance", "reliability"]
---

Redis is often introduced as “just a fast key-value store”.

That’s misleading.

Redis is a **single-threaded execution engine with strong atomic guarantees** — and Lua is how you *use that power properly*.

This post explains **when and why Redis + Lua matters**, using a real example:  
*looping over keys and deleting them safely without races*.

No magic.  
No micro-optimizations.  
Just honest behavior under concurrency.

---

## The problem: multi-step logic on shared state

Let’s say you want to:

- find a set of keys
- apply some condition
- delete them
- maybe return how many were removed

The naive approach looks like this:

```python
keys = redis.scan("session:*")
for k in keys:
    if should_delete(k):
        redis.delete(k)
````

It works.

Until it doesn’t.

### What goes wrong?

* Another client modifies keys mid-loop
* A key disappears between `SCAN` and `DEL`
* You partially complete the operation
* Retries make things worse

This isn’t a Redis problem.

It’s a **client-side atomicity problem**.

---

## Redis guarantees atomic execution — but only per command

Redis is single-threaded.

That means:

* each command runs atomically
* but **multiple commands are not grouped automatically**

From Redis’ point of view, this:

```
SCAN → GET → DEL
```

is three unrelated operations.

If you want atomic *logic*, you must move the logic **into Redis**.

That’s where Lua comes in.

---

## Lua scripts: atomic logic, not just faster code

A Redis Lua script runs:

* on the Redis server
* in a single execution
* without interleaving with other commands

From the outside, it behaves like **one atomic operation**.

This is the key mental shift:

> Lua is not about speed.
> Lua is about **correctness under concurrency**.

---

## A concrete example: loop + delete, safely

### Goal

Delete all keys matching a pattern **atomically**
and return how many were deleted.

### Lua script

```lua
local cursor = "0"
local deleted = 0

repeat
  local result = redis.call("SCAN", cursor, "MATCH", KEYS[1], "COUNT", 100)
  cursor = result[1]
  local keys = result[2]

  for _, key in ipairs(keys) do
    redis.call("DEL", key)
    deleted = deleted + 1
  end
until cursor == "0"

return deleted
```

### Invocation

```python
redis.eval(lua_script, 1, "session:*")
```

### What you get

* One atomic operation
* No race conditions
* No partial state
* No retry ambiguity

If it runs, it runs fully.

If it fails, nothing halfway happened.

---

## Why not use MULTI / EXEC?

Transactions help — but they don’t solve everything.

### MULTI / EXEC limitations

* Commands are queued, not executed immediately
* Conditional logic still happens client-side
* You still fetch → decide → write

Lua lets you:

* read
* compute
* mutate
* return results

All **inside Redis**.

---

## Why this matters under load

Under concurrency:

* client-side loops amplify race windows
* retries multiply traffic
* partial updates become permanent bugs

Lua scripts:

* reduce round-trips
* shrink race windows to zero
* make behavior deterministic

> Deterministic systems are debuggable systems.

---

## Where people go wrong with Redis + Lua

### 1. Treating Lua as “business logic”

Don’t.

Lua scripts should:

* manipulate Redis data
* enforce invariants
* perform atomic state transitions

They should **not** encode domain rules.

Think “transaction”, not “service”.

---

### 2. Forgetting that Lua blocks Redis

Lua scripts run on Redis’ main thread.

That means:

* long scripts block all clients
* large loops can stall the server

Rules of thumb:

* keep scripts short
* limit iterations
* avoid unbounded loops
* batch when needed

Atomic doesn’t mean infinite.

---

### 3. Passing dynamic code instead of parameters

Bad idea:

```python
redis.eval(f"redis.call('DEL', '{user_input}')", 0)
```

Good idea:

* static scripts
* dynamic data via KEYS / ARGV

This keeps scripts safe, cacheable, and predictable.

---

## Redis + Lua fits a specific mindset

Use Redis + Lua when:

* you need **atomic multi-step updates**
* correctness matters more than convenience
* retries must not duplicate work
* you want predictable behavior under concurrency

Avoid it when:

* logic belongs in your application
* operations are long-running
* data volume is unbounded

---

## The bigger picture

Redis + Lua is part of a broader philosophy:

* **move invariants close to the data**
* **make race conditions impossible, not unlikely**
* **prefer explicit atomicity over hopeful sequencing**

It’s the same thinking behind:

* database transactions
* compare-and-swap
* idempotent APIs
* event versioning

---

## Final takeaway

Redis Lua scripts aren’t a performance hack.

They’re a **correctness tool**.

If your system depends on:

* shared mutable state
* concurrent updates
* retries under failure

then Redis + Lua gives you something invaluable:

> A way to say “this happens — or nothing happens.”

And in production systems, that honesty is worth more than raw speed 😛

---

## Related reading

* Redis documentation: Lua scripting and atomicity
* Idempotency and retry-safe APIs
* Why client-side loops fail under concurrency
* Designing state machines with Redis
