# Rate-Limiting — Level 2: Trade-offs

## What's good
- **Boundary enforced by the compiler**, not discipline — consumers must use
  `RateLimitApi`; the engine and registries cannot leak.
- **Swappable store** — in-memory ↔ Redis is a one-line adapter change.
- **First-class extensibility** — `RateLimitAlgorithm` is a port, so a client ships a
  custom algorithm without forking the package.
- **Deterministic algorithm tests** — a fake `Clock` + in-memory store exercise refill /
  window behaviour with no real time and no infrastructure.
- **Clear contract** — `RateLimitApi` is explicit and stable.

## What it costs
- **More ceremony** — ports, adapters, a composition root; overkill for a tiny app (YAGNI).
- **Indirection** — one more hop to read when tracing a call.
- **No deployment win** — runtime is still one process; a **correct cross-instance counter
  still needs a shared-store adapter** (or Level 3). You bought a clean boundary and
  testability, not distribution.

## When Level 2 is the right choice
- The component is shared across apps/teams, you want the boundary locked, you want the
  algorithms unit-tested, and you may need to **swap the store** — but you do **not** yet
  need a distributed counter.
- Often the **best long-term home**: many systems never need Level 3.

## What would push you to Level 3
- A **global** limit across many instances or services: the counter must live in a
  **shared atomic store** (or go approximate), and you must decide **where** the limiter
  runs (in each service vs the gateway vs a sidecar) and what happens when the store is
  down.

## Carry-forward to Level 3
- The **`CounterStore` port** is where a shared/distributed store plugs in — and where
  atomicity moves from an app-side lock to a **store-side atomic script**.
- The **`RateLimitAlgorithm` port** is where a **distribution-friendly (approximate)**
  algorithm can be added — trading exactness for scale — with no engine change.
