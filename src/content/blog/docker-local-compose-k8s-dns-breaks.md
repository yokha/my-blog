---
title: "From Localhost to Kubernetes: How DNS Really Works (and Why It Breaks)"
description: "A practical mental model of name resolution across local development, Docker, Docker Compose, and Kubernetes — and why DNS issues often appear only in production."
pubDate: 2026-01-27
tags: ["docker", "kubernetes", "dns", "containers", "nss", "backend", "production"]
---

# From Localhost to Kubernetes: How DNS *Really* Works (and Why It Breaks)

When things work locally but fail in containers or Kubernetes, the root cause is often not networking.

It’s **name resolution**.

Understanding how DNS and hostname resolution evolve from:

* local development
* Docker Compose
* Kubernetes

is one of those backend fundamentals that quietly saves you days of debugging.

This post builds a **single mental model** that explains what’s happening — and why assumptions break as you scale.

---

## Stage 1: Local Development — “localhost lies to you”

On your laptop, life is simple:

```text
localhost → 127.0.0.1
```

Your application:

* talks to `localhost`
* opens a socket
* connects immediately

Resolution path (simplified):

```
app → libc → /etc/hosts → DNS
```

And `/etc/hosts` usually contains:

```
127.0.0.1   localhost
```

### The hidden assumption

> “If it works on localhost, the name is correct.”

This assumption **dies the moment you containerize**.

---

## Stage 2: Docker Containers — localhost now means “me”

Inside a container:

```text
localhost → the container itself
```

Not your laptop.
Not another service.

### Classic failure

```text
DB_HOST=localhost
```

Works locally ❌
Fails in Docker ❌
Silently connects to nothing ❌

### Docker DNS model

Docker gives you:

* an **internal DNS**
* service-name-based resolution
* isolated network namespaces

In Docker Compose:

```yaml
services:
  api:
    depends_on: [db]
  db:
```

You **must** use:

```text
DB_HOST=db
```

Resolution becomes:

```
api container → Docker DNS → db container IP
```

### Key rule

> In containers, **service names replace hostnames**, not IPs.

---

## Stage 3: Docker Compose — implicit DNS magic

Docker Compose feels easy because it hides complexity:

* Each compose project creates a virtual network
* Each service name becomes a DNS A record
* Containers auto-register on startup

Example:

```text
api → db → redis → worker
```

DNS inside the network:

```
db       → 172.x.x.x
redis   → 172.x.x.x
worker  → 172.x.x.x
```

### Why this works so well

* DNS is **dynamic**
* Containers can restart
* IPs change safely
* Names stay stable

This is your **first exposure to service discovery** — whether you realize it or not.

---

## Stage 4: Kubernetes — DNS becomes a first-class system

Kubernetes doesn’t just *support* DNS.

It **depends on it**.

### The Kubernetes DNS contract

Every Service gets:

```
<service>.<namespace>.svc.cluster.local
```

Example:

```
postgres.default.svc.cluster.local
```

Most clients just use:

```
postgres
```

because:

* namespace search paths are injected automatically
* `/etc/resolv.conf` is cluster-aware

### Resolution path in Kubernetes

```
app → libc → NSS → CoreDNS → Service → Pod endpoints
```

This is where things get interesting.

---

## Where things start breaking: NSS and hostname resolution

At scale, **DNS is not just DNS**.

It’s:

* libc
* NSS (`/etc/nsswitch.conf`)
* hostname libraries
* container base image choices

### Real-world failure mode

You see errors like:

```
Temporary failure in name resolution
```

Or:

```
getaddrinfo() failed
```

Even though:

* the service exists
* CoreDNS is healthy
* `nslookup` works

### Why?

Because **applications don’t all resolve names the same way**.

Some go through:

* glibc + NSS
* systemd-resolved
* musl (Alpine)
* static binaries (Go!)

---

## Go vs Python vs Java (briefly)

### Go

* Often uses **pure Go DNS resolver**
* May bypass NSS entirely
* Behavior depends on:

  * CGO enabled or not
  * base image (glibc vs musl)

### Python / Java

* Go through libc
* Obey `/etc/nsswitch.conf`
* Affected by missing `hosts: files dns`

### Alpine gotcha

Alpine uses **musl**, not glibc.

Common symptoms:

* DNS works in one image, not another
* Hostname resolution behaves differently
* NSS expectations don’t match reality

---

## The hidden config file nobody checks

Inside containers:

```text
/etc/nsswitch.conf
```

This controls resolution order:

```
hosts: files dns
```

If misconfigured:

* `/etc/hosts` may be ignored
* DNS queries may never fire
* resolution fails silently

> Kubernetes assumes sane NSS behavior — your image might not provide it.

---

## Kubernetes adds another twist: Pod hostnames

Kubernetes sets:

* `hostname`
* `subdomain`
* pod DNS entries (optional)

But **applications rarely need pod-level DNS**.

They should:

* talk to Services
* not individual Pods
* avoid hostname-based assumptions

---

## The big mental model

### Evolution of name resolution

| Environment | What names mean                  |
| ----------- | -------------------------------- |
| Local       | `/etc/hosts`                     |
| Docker      | Service name → container         |
| Compose     | Service name → dynamic IP        |
| Kubernetes  | Service name → virtual IP → pods |

### What breaks systems

* Hardcoding `localhost`
* Using IPs instead of names
* Assuming DNS == `/etc/hosts`
* Ignoring NSS behavior
* Mixing base images blindly

---

## Practical rules that survive all environments

1. **Never use `localhost` between services**
2. **Always use service names**
3. **Avoid IPs**
4. **Understand your base image DNS stack**
5. **Test resolution inside the container**
6. **Assume restarts, reschedules, and IP churn**

---

## Why this matters for production resilience

When DNS fails:

* services can’t start
* retries amplify load
* health checks flap
* cascading failures begin

Most “network issues” are actually:

> name resolution mismatches across environments

---

## Final takeaway

Docker and Kubernetes didn’t make networking harder.

They made it **explicit**.

If you treat DNS and hostname resolution as part of your system design — not an afterthought — your services become:

* more portable
* more debuggable
* more predictable under failure

And most importantly:

> They stop working *only* where you expect them to 😛
