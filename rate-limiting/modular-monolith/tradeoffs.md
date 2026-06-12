# Rate-Limiting — Level 1: Trade-offs

## What's good
- **Simplicity:** a check is a function call + a map lookup — no network, no serialization.
- **Fast:** an in-process counter adds almost nothing to the hot path.
- **Atomic (in one process):** a per-key lock makes the read-modify-write safe.
- **Easy to reason about:** one process, one map, one place to debug.

## What hurts (the limits)
- **In-process counter breaks horizontal scaling — the killer.** N instances → N
  independent counters → the effective limit is **×N**. Rate-limiting hits this wall the
  moment you run a second instance, far earlier than Auth.
- **Lock contention on hot keys:** a heavily-hit key (one noisy IP) serializes on its lock.
- **State is volatile:** an in-process map is lost on restart — counters reset (often
  acceptable, effectively fail-open, but it *is* a reset).
- **Eviction is your job:** without native TTL you must sweep expired keys or leak memory.

## When Level 1 is the right choice
- A **single instance**, or a dev/simple app.
- **Per-instance approximate** protection is good enough (crude abuse damping, not a hard
  global quota).
- Otherwise, the realistic Level-1 deployment is already **in-process engine + shared
  cache** for the counter — which is structurally Level 3's counter, reached early.

## What would push you up a level
- **→ Level 2 (module):** you want the `CounterStore` (and algorithm) behind an enforced
  port so you can swap in-process ↔ Redis and **unit-test the algorithms** with a fake
  clock — without touching the engine.
- **→ Level 3 (microservice / shared):** you need a **global** limit across many instances
  or services, so the counter must live in a **shared atomic store** (or go approximate),
  and you must decide where the limiter runs (in-app vs gateway vs sidecar).

## Carry-forward to the next levels
- **`CounterStore` is the seam.** Swapping in-process map → shared atomic store is the
  central move of the whole ladder.
- The **`RateLimitAlgorithm` interface + injected `Clock`** are already the test seams
  Level 2 exploits (deterministic algorithm tests with no real time, no infra).
- **Atomicity migrates** from an app-side lock (L1) to a **store-side atomic script** (L3)
  — same guarantee, different owner.
