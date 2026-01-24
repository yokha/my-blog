---
title: "Monoliths, Multi-Tenancy, and the Inevitable Split"
description: "Starting with a monolith is fine. What matters is designing it so that splitting later is possible, safe, and boring."
pubDate: 2026-01-24
tags: ["architecture", "backend", "fastapi", "multi-tenancy", "distributed-systems"]
---

Most backend systems don’t start complicated.

You begin with a single FastAPI service.

It talks to a database.

Maybe it routes requests based on a tenant ID.

Maybe it proxies some calls to other services.

It works.

It ships.

It grows.

And then, slowly at first, the questions change.

---

## The questions that signal growth

At some point, you start hearing things like:

- “We need to onboard five more tenants — each in a different region.”
- “Can this service route requests based on tenant configuration from the database?”
- “How do we scale the backend logic without scaling the proxy layer?”
- “Can we isolate tenant failures from each other?”

None of these questions are unreasonable.

They’re signs that your system is **successful**.

They’re also signs that the architecture is about to be stressed.

---

## When everything lives in one box

What started as a clean API slowly becomes a mix of concerns:

- request routing
- tenant discovery
- region-aware dispatching
- business logic
- service-to-service proxying
- retries and fallback paths

All in one process.
All in one deployable unit.

At first, this feels efficient.
Later, it feels heavy.

---

## The hidden costs of a growing monolith

When too many responsibilities live together, the pain shows up quietly:

- **Testing becomes harder**  
  You can’t test routing logic without touching business logic.

- **Scaling becomes coarse-grained**  
  You scale everything or nothing.

- **Security boundaries blur**  
  Proxy logic and business logic share the same blast radius.

- **Change becomes risky**  
  Small changes require redeploying everything.

None of this means you “chose the wrong architecture”.

It means the architecture wasn’t designed with change in mind.

---

## Start simple — but design for refactorability

Starting with a monolith is not a mistake.

In fact, it’s often the right choice.

The mistake is **writing a monolith that cannot be split**.

Even when everything runs in the same process, you can still:

- separate concerns cleanly
- define explicit boundaries
- use interfaces between components

> Refactorability is a feature.

---

## Practical design rules that age well

Even in a single FastAPI service, a few decisions make a huge difference later.

### 1. Keep proxy logic separate

Tenant-aware routing, retries, and fallbacks should not live inside business handlers.

Treat them as a layer:
- a dispatcher
- a gateway
- a routing component

Even if it’s just a Python module today.

---

### 2. Isolate tenant resolution

Resolving:
- tenant identity
- configuration
- region
- feature flags

…should happen **before** business logic runs.

This keeps your core logic:
- testable
- tenant-agnostic
- easier to extract later

---

### 3. Use interfaces, even inside one process

If two parts of your service communicate, give them an interface.

Not for abstraction purity — but for future extraction.

> If you can replace a component with a fake in tests, you can replace it with a service later.

---

## The inevitable split (when things go well)

If growth continues, the architecture you *prepared* for starts to emerge naturally:

- **Dispatcher Service**  
  Handles tenant-aware routing, proxying, retries, and region selection.

- **Worker Service**  
  Owns business logic, domain rules, and data access.

- **Tenant / Config Service** (optional)  
  Centralizes tenant metadata and configuration.

Because you designed with separation in mind:
- the split is mechanical
- not emotional
- not a rewrite

---

## A real-world multi-tenant lesson

Multi-tenant proxy logic is a great example.

Early on:
- putting it directly in code is fine
- testing with two tenants is easy

Later:
- cloud-native platforms (Kubernetes, Swarm)
- regional deployments
- partial scaling
- independent security requirements

Suddenly, the same code becomes a liability.

Unless you designed it to move.

---

## The real takeaway

You don’t need microservices on day one.

You don’t need service meshes.
You don’t need ten repos.

But you **do** need to assume that success brings complexity.

> Architect for change, not for scale.

Embrace simplicity.
Ship a monolith.
Just don’t trap yourself inside it.

Because if things go well,  
you’ll be very glad you left the door open.

---

### Related reading

- REST to Event-Driven: When to Split Your Backend
- Async Doesn’t Make Your System Fast — It Makes It Honest
- Python Async Made Simple: Process vs Thread vs Coroutine
- FastAPI Swagger: Auto-Magic… Until You Need Control
