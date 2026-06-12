# Rate-Limiting — Component Design

An **infra component** that throttles actions by **named policies**, keyed by a dimension
(IP, identity, route, …), returning **allow / deny + retry metadata**. Consumed by other
components (Auth references it for login / register / logout / forgot-password).

> Design only — docs, UML (Mermaid), and language-agnostic pseudocode. No runnable code.

## Scope

- **In:** check (consume), compose (AND), peek, clear-on-success, define-policy (config),
  register-algorithm (extension). Per-policy fail-open / fail-closed.
- **Out / deferred:** usage / billing quotas, concurrency (in-flight) limiting, distributed
  quota reconciliation, dynamic policy-config service / UI.

## How to read it

```
base/   →   modular-monolith/   →   module/   →   microservice/
(WHAT,        (L1: in-process)       (L2: ports    (L3: shared atomic
 level-                              & adapters)    store / edge)
 independent)
```

The **`base/` requirements never change** — each level only changes *where the counter
lives* and *how the atomic update is done*. Read `base/` first, then the levels in order.

## base/ — the level-independent contract

| File | What it holds |
|---|---|
| [context.md](base/context.md) | environment, consumers, the policy/key/decision model, the algorithm seam, the atomicity concern |
| [user-stories.md](base/user-stories.md) | check / compose / clear / define-policy / plug-in-algorithm / peek + NFRs |
| [use-case.md](base/use-case.md) | use-case diagram |
| [crc-cards.md](base/crc-cards.md) | collaboration board (card-style) + CRC cards |
| [domain-model.md](base/domain-model.md) | class diagram centered on the `RateLimitAlgorithm` seam + per-algorithm state |
| [data-model.md](base/data-model.md) | key → value state per algorithm; namespacing; TTL; why it's a cache, not a ledger |
| [glossary.md](base/glossary.md) | ubiquitous language |

## The three levels (the 1→3 thread)

The counter **is** shared state — so the journey is the **inverse of Auth's**: not "escape
the shared store", but "keep a shared counter correct as you distribute".

| Level | Counter lives | Atomicity | Correct across instances |
|---|---|---|---|
| [1 — modular-monolith](modular-monolith/architecture.md) | in-process map (or shared cache) | lock / atomic op in-process | only via a shared store |
| [2 — module](module/architecture.md) | behind a `CounterStore` port | same, swappable adapter | same |
| [3 — microservice](microservice/architecture.md) | shared atomic store (Redis) / edge | store-side atomic script, or **approximate** local counters | **yes** |

Each level has the same four files: `architecture` · `sequence` · `pseudocode` · `tradeoffs`.
Full comparison: [microservice/tradeoffs.md](microservice/tradeoffs.md).

## Key design decisions

- **Algorithm is a pluggable interface** (`RateLimitAlgorithm`) selected by name from a
  registry — token bucket / sliding-window / fixed-window ship built-in, and a client can
  register its own with **no engine change** (OCP).
- **Algorithm = pure decision logic; `CounterStore` = atomic persistence.** Atomicity is a
  separate, explicit concern, not smeared into the algorithm.
- **General API `check(policy, key)`** — Auth's two-bucket `check(identity, ip)` is just
  two AND-ed policy checks (`login-per-identity` + `login-per-ip`), `retryAfter` = max.
- **Per-policy fail-open / fail-closed** — security-sensitive limits (login) can deny on
  store failure; public reads can allow.
- **Bounded state** — every key carries a TTL; the store is a cache, not a ledger.

## Status

Complete across `base/` + all three levels. The 1→3 thread is the **inverse of Auth's**:
the counter is shared state, so the journey is keeping it atomic and fast as you
distribute — in-process lock → swappable `CounterStore` port → shared store (store-side
script / CAS) or approximate local counters.
