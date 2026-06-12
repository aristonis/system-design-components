# Rate-Limiting — Level 3: Trade-offs

## What's good (why pay for this)
- **Global correctness** — the limit holds across every instance and service, not per
  process.
- **Placement choice** — coarse limits at the gateway, fine-grained in services, or
  offloaded to a sidecar.
- **Scale options** — exact central store, or approximate local counters when a per-check
  hop is too expensive.

## What it costs
- **A network hop per check** (central) on the request hot path — latency you did not pay
  at L1/L2.
- **The store is a bottleneck / SPOF** — it needs HA + sharding by key; a mirror of Auth's
  "call auth per request."
- **Store-side scripts duplicate algorithm logic** — the pure, pluggable algorithm story
  weakens for that path; CAS avoids this but adds retries.
- **Approximate counters overshoot** — exactness traded for scale.
- **Operational weight** — HA store, sharding, placement decisions, monitoring, fail-mode
  tuning per policy.

## When Level 3 is justified
- A **real global limit** across a fleet: abuse protection at scale, multi-service quotas,
  or a shared budget that must hold no matter how many instances run. Without that, an
  in-process limiter (L1) or a shared-cache adapter behind the L2 port is enough.

## What you lose vs the monolith
- The **free in-process counter** and the **zero-hop** check.
- **Exactness for free** — you now choose exact-but-centralized or approximate-but-scalable.

---

## The three levels at a glance (the journey)

| Aspect | L1 Modular Monolith | L2 Module (ports & adapters) | L3 Shared / Distributed |
|---|---|---|---|
| Counter location | in-process map | behind `CounterStore` port | shared atomic store (or approximate local) |
| Atomicity | per-key lock | port (lock / CAS) | store-side script **or** CAS retry |
| Correct across instances | no | no (unless shared adapter) | **yes** |
| Check cost | function call | function call | network hop (central) / local (approx) |
| Exactness | exact (single instance) | exact (single instance) | exact-ish (central) / overshoot (approx) |
| Extensible algorithm | registry | port | pure via CAS / script duplicates logic |
| Placement | in-app | in-app | in-app lib / gateway / sidecar |
| Complexity | low | medium | high |
| Best for | single instance / approximate | shared, testable, swappable store | a real global limit across a fleet |

**The lesson (inverse of Auth):** the counter **is** shared state, so the journey is not
"escape the shared store" but "keep a shared counter **atomic and fast** as you
distribute." That need shows up early — at the **second instance** — and each rung trades
simplicity for reach: an in-process lock becomes a store-side script; an exact local count
becomes an exact-but-centralized or approximate-but-scalable global count. Climb only when
a **global** limit forces it.
