---
title: "Vault is Not One Thing: KV, PKI, Automation — and Why Mixing Them Hurts Systems"
description: "Vault is Not One Thing: KV, PKI, Automation."
pubDate: 2026-02-08
tags: ["vault", "pki", "security", "devops", "ansible", "architecture"]
draft: false
---

## Vault is Not One Thing  
### KV, PKI, Automation
When people say *“we use Vault”*, the sentence is incomplete.

Vault is not one thing.  
It is **a platform with very different operational personalities**, and confusing them leads to fragile systems, unnecessary coupling, and security debt.

This post is about **distinguishing Vault use-cases**, with a deliberate focus on **Vault PKI**, why it exists, why it’s hard, and why it should *not* be treated like a runtime key/value database.

> I care about this distinction because **security systems fail silently** — until they don’t.

---

## The Three Faces of Vault

Let’s name things properly.

### 1. Vault (the product)

Vault is a *security control plane*.

At its core, Vault provides:
- authentication
- authorization
- secret materialization
- lifecycle control (issue, rotate, revoke)

Vault is **stateful**, **audited**, and **opinionated** about access.

> Vault is not designed to be on the hot path of your application.

---

### 2. Vault KV (Key/Value secrets)

Vault KV is often the first thing teams touch:
- database passwords
- API tokens
- credentials injected at startup

This works **well** when:
- secrets are read **at boot**
- refreshed **infrequently**
- cached locally by the application

It works **poorly** when:
- used as a runtime config store
- polled per request
- treated like Redis or Consul

**Why?**

Because Vault optimizes for **security and correctness**, not latency or throughput.

KV is a **delivery mechanism**, not a data plane.

---

### 3. Vault PKI: what people underestimate?

Vault PKI turns Vault into:
- a **Certificate Authority**
- a **trust anchor**
- a **policy-driven issuer**

> This is not “just another secret”.

---

## Why PKI Is a Different Problem Entirely

Passwords are *shared secrets*.  
Certificates are *identity claims*.

That difference changes everything.

With PKI, you must answer:
- Who is allowed to get a certificate?
- For which identity?
- With which TTL?
- Signed by which CA?
- How do clients verify trust?
- What happens on compromise?

> Vault PKI gives you **structured answers** to those questions.
But only if you **respect its role**.

---

## Vault PKI Is Not Runtime Infrastructure

> Vault PKI should **never** sit in your request path.

Correct mental model:

```text
Vault PKI = factory
Certificates = products
Applications = consumers
````

Flow:

1. Vault issues certificates
2. Certificates are stored locally (disk, keystore, secret volume)
3. Runtime systems use certificates **without Vault involvement**

If Vault goes down, **your system should keep running**.

This assumes certificates are issued with sensible TTLs and renewed **before** expiry — not at the last minute.

---

## Revocation Reality (and Why TTLs Matter More)

Revocation is where PKI architecture shows its true cost.

Revoking a certificate does not instantly remove trust from the system.
It relies on:

* certificate TTLs
* client-side caching
* renewal behavior
* protocol semantics

This is another reason Vault must not sit in the runtime path: revocation is a **lifecycle and policy concern**, not a request-time operation.

> Short-lived certificates reduce blast radius far more reliably than “instant revocation” expectations.

---

## Why Using Vault as Runtime KV Is a Smell

Using Vault KV as a live configuration database usually means:

* unclear ownership of config
* lack of versioning discipline
* runtime coupling to security infrastructure
* accidental DoS on Vault
* fear-driven “secure everything” thinking

> Security tools should **reduce blast radius**, not expand it.

---

## Vault + Ansible: Control Plane, Not Magic

Vault paired with Ansible is powerful — but only when roles are clear.

What Ansible should do:

* provision Vault
* configure PKI backends
* define roles and policies
* bootstrap trust material
* enforce reproducibility

What Ansible should not do:

* fetch secrets per task dynamically at scale
* replace proper secret injection mechanisms
* blur infra vs runtime boundaries

> Vault + Ansible is about **determinism**, not convenience.

---

## Vault PKI Requires HA — But Not Runtime HA

Vault PKI is critical infrastructure, but not *runtime* infrastructure.

That distinction matters when designing high availability.

### What must be highly available

Vault PKI must be:

* available during **certificate issuance**
* available during **rotation windows**
* available during **bootstrapping and onboarding**
* available during **incident recovery**

> In other words: **control-plane availability**, not data-plane availability.

If Vault is temporarily unavailable:

* TLS handshakes must still work
* running services must continue operating
* existing certificates must remain valid

> If that is not true, the architecture is already broken.

---

## Vault + Raft: Why This Combination Makes Sense

Vault’s integrated storage using **Raft** is often misunderstood.

Raft is not there for scale.
Raft is there for **consistency and correctness**.

With Raft-backed Vault:

* state is replicated across Vault nodes
* leadership is explicit
* writes are serialized and audited
* failure modes are predictable

This matches the **security-first nature** of PKI:

* issuing a certificate is a *decision*, not a cache lookup
* revocation must be consistent
* audit logs must be authoritative

> For Vault PKI, **Raft is a feature, not a limitation**.

You are trading:

* horizontal scale ❌
  for:
* correctness
* auditability
* deterministic behavior

That is the right trade.

---

## HA Model That Actually Works for Vault PKI

A sane Vault PKI HA setup looks like this:

* 3 Vault nodes
* Raft integrated storage
* clear leader election
* TLS everywhere
* clients talking to a logical Vault endpoint (DNS / LB)

Importantly:

* clients do **not** retry aggressively
* applications do **not** depend on Vault per request
* certificate issuance is rate-limited and intentional

> Vault outages should be **operational events**, not customer incidents.

---

## Why Containerising Vault PKI Often Adds No Value

> This is where opinions start — but they’re earned.

For Vault PKI, containerisation usually adds:

* another abstraction layer
* more failure modes
* more moving parts
* harder debugging
* zero security benefit

Vault PKI:

* is stateful
* depends on disk durability
* depends on stable identity
* depends on predictable restarts

Running it as:

* a systemd-managed service
* on dedicated VMs or hosts
* with explicit storage paths

is often **simpler, safer, and easier to reason about**.

Containers shine when:

* workloads are stateless
* restarts are cheap
* scale is elastic

Vault PKI is none of those.

---

## The Real Reason: Don’t Couple Vault’s Lifecycle to Your App Runtime

One of the main reasons I avoid containerising Vault PKI is not performance or ideology — it’s **failure-domain coupling**.

If Vault lives inside the same Docker Compose / Swarm stack as the applications it enables, you’ve silently linked their lifecycles:

* `stack deploy` restarts your runtime → Vault gets touched
* a bad app rollout → service reconciliation → Vault can bounce
* a node drain meant for “apps” → Vault moves too
* “we’re just restarting the stack” → you accidentally restart *trust*

That’s backwards.

Vault PKI should be a **separate control-plane dependency** with its own:

* upgrade cadence
* restart policy
* storage discipline
* incident handling
* blast radius

> If “redeploying the app stack” can restart your CA, you don’t have a platform — you have a shared fate.

---

## When Containers *Do* Make Sense (Rarely)

Containerising Vault can still make sense when:

* you already have strong operational maturity
* storage is explicitly managed (not ephemeral)
* identity is stable
* restart behavior is controlled
* debugging is understood

This should be a **deliberate choice**, not a default.

> If the reason is *“everything else is containerised”*, that’s not architecture — that’s habit.

---

## Common Vault PKI Anti-Patterns

* Vault queried per request
* Certificates issued with extremely long TTLs “to avoid renewal issues”
* Vault running inside the same deployment stack as applications
* Treating Vault KV as a runtime config database
* Restarting Vault as part of application rollouts

---

## Strengths of Vault PKI

* Centralized trust
* Short-lived certificates
* Automated rotation
* Clear audit trail
* Policy-driven issuance

---

## Weaknesses (That Matter)

* PKI is complex (no way around it)
* Debugging TLS failures is hard
* Misconfigured trust chains break everything
* Operational discipline is required

---

## The Mental Model That Actually Works

| Layer          | Responsibility         |
| -------------- | ---------------------- |
| Vault PKI      | Identity & trust       |
| Certificates   | Runtime authentication |
| Applications   | Business logic         |
| Runtime config | App-local / cached     |
| Observability  | Independent            |

---

## Final Thought

Vault is not a database.
Vault is not a cache.
Vault is not your config service.

Vault is a **trust system**.

When you treat it as such — especially with PKI — it becomes one of the strongest building blocks in a modern architecture.

Happy reading!