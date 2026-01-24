---
title: "Evolving a FastAPI Backend: REST, WebSockets and Event-Driven"
description: "When a FastAPI backend grows beyond REST: how messaging, WebSockets, and event-driven UX change your architecture — and how to avoid common traps."
pubDate: 2026-01-24
tags: ["fastapi", "backend", "event-driven", "websockets", "architecture", "async"]
---

Most FastAPI backends start the same way.

A clean REST API.  
A frontend calling it.  
Maybe some external clients.

It works well — until it doesn’t.

As systems grow, requirements change:
- background processing becomes normal
- async messaging enters the picture (Kafka, Redis Streams, NATS…)
- the frontend needs real-time updates
- REST endpoints start producing events instead of final results

At that point, an architectural question appears:

> Should the backend handle everything or is it time to split responsibilities?

This post walks through that transition, the trade-offs involved, and a pattern that keeps both **scalability** and **user experience** intact.

---

## The starting point: a classic FastAPI backend

The initial architecture is simple:

```

Browser / Client
    |
   REST
    |
FastAPI Backend

```

FastAPI:
- handles HTTP requests
- performs work
- returns a response

Even when async, the model is still request -> response.

This simplicity is a feature (remind me Golang ;) ) until you introduce **asynchronous work**.

---

## Growth changes the rules

As the system evolves, new needs emerge:

- long-running operations
- integration with other services
- B2B workflows via event streams
- live updates in the browser

You introduce messaging:

```
FastAPI -> Kafka / Redis Streams /  / ...
```

Now REST endpoints don’t *finish* the work but they **emit events**.

The frontend still wants feedback.  
The backend no longer owns the full lifecycle.

> This is where architecture decisions start to matter.

---

## Option 1: Keep everything inside FastAPI

One approach is to keep all responsibilities in a single service.

```

FastAPI
├─ REST endpoints
├─ Pub/Sub producer
├─ WebSocket server
└─ Background tasks

```

### Why this is tempting

- simple deployment
- fewer moving parts
- fast to prototype

### Where it starts to hurt

- WebSockets don’t scale cleanly with Gunicorn workers
- background tasks and WebSockets increase coupling
- shared state becomes fragile
- cloud-native scaling becomes awkward

You end up with one service trying to be:
- an API
- an event producer
- a real-time delivery layer
- ...

> This works until load, failures, or complexity increase.

---

## Option 2: Split FastAPI and delivery concerns

A more scalable approach is to separate responsibilities explicitly.

```
Browser
    ↑
WebSocket Gateway
    ↑
Message Broker
    ↑
FastAPI Backend
```

### Clear ownership

**FastAPI backend**
- handles REST
- validates input
- performs core business logic
- produces events

**WebSocket gateway**
- consumes messages
- manages client connections
- pushes updates to browsers

### Benefits

- clean separation of concerns
- independent scaling
- pub/sub becomes a first-class design choice
- simpler operational boundaries

This architecture aligns naturally with event-driven systems.

But it introduces a new problem.

---

## The fire-and-forget UX problem

In this model:

1. Browser calls a REST endpoint
2. Backend publishes an event
3. REST returns `200 OK`
4. Browser *waits* for a WebSocket message… maybe

What happens if:
- the message is delayed?
- the client reconnects?
- something fails silently?

You lose the immediate feedback loop.

This affects:
- user experience
- error visibility
- debuggability

> The system *works*, but the UX feels unreliable.

---

## Async makes system behavior visible

If this feels uncomfortable, that’s expected.

Async systems don’t hide latency, contention, or failure but they expose them.
A REST endpoint returning immediately while work continues elsewhere forces you
to confront how your system actually behaves.

This is the same idea behind async programming itself:
it doesn’t make systems faster by default. It makes waiting explicit.

Once you accept that, designing around event-based state becomes natural.

---

## Closing the loop: event-based state, not assumptions

To fix the UX problem, you need to make **state explicit**.

A simple pattern:

1. REST endpoint returns a **job / operation ID**
2. Backend emits events tied to that ID
3. WebSocket pushes updates by ID
4. Frontend listens or correlates state

```

POST /action -> job_id
    |
Message Broker
    |
WebSocket Gateway
    |
Browser (by job_id)

```

Now:
- success and failure are observable
- long-running work is traceable
- UX reflects *state*, not  :)

> You’re no longer waiting for a message but you’re tracking progress.

---

## Reference architecture (simplified)

**REST-driven model**

```

Client
|
HTTP
|
Backend

```

**Event-driven model**

```
Client
 |
REST
 |
Backend -> Message Broker -> Consumers
 └─► WebSocket Gateway -> Client
```

> Think of REST as *command entry* and messaging as *state propagation*.

---

## When not to do this

Not every FastAPI backend needs messaging, gateways, or event-driven UX.

You probably **don’t** need this architecture if:

- requests are short-lived and synchronous
- the frontend only needs immediate REST responses
- failures are rare and easy to retry inline
- you’re early-stage and optimizing for speed of change

In those cases, a simple REST + async backend is often the best choice.

> Architecture should follow pressure, not fashion.

Split responsibilities when:
- async workflows become the norm
- user feedback depends on background processing
- observability and failure handling start to matter more than simplicity

> Until then, keep it boring ;)

---

## Final guidance

If you’re introducing pub/sub into a FastAPI system:

- don’t just bolt it onto REST
- refactor with a clear separation
- design delivery as its own concern

Let FastAPI focus on **core logic and event production**.  
Let a gateway focus on **real-time delivery**.  
Track state explicitly to **close the UX loop**.

> Async systems reward clarity.

> Design for it early: your users (and future you) will feel the difference.
---
