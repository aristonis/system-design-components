# Rate-Limiting — Level 3: Pseudocode

The **engine, policies, and the pure algorithms are unchanged**. What is new: the
`CounterStore` becomes **shared and network-backed**, and atomicity moves to either a
**store-side script** or **optimistic CAS**.

## Option A — store-side atomic script (token bucket)

```
script TOKEN_BUCKET(key, cap, rate, now, ttl):     // runs atomically on the store
    s = GET(key) or { tokens: cap, lastRefillAt: now }
    tokens = min(cap, s.tokens + (now - s.lastRefillAt) * rate)
    if tokens >= 1:
        SET(key, { tokens: tokens - 1, lastRefillAt: now }, EX = ttl)
        return { allowed: true,  remaining: floor(tokens - 1), retryAfter: 0 }
    else:
        SET(key, { tokens: tokens, lastRefillAt: now }, EX = ttl)
        return { allowed: false, remaining: 0, retryAfter: (1 - tokens) / rate }

class ScriptCounterStore implements CounterStore:
    atomicApply(key, ttl, fn):
        // fn is not run client-side here; the script IS the decide, server-side
        return store.eval(scriptFor(policy.algorithm), key, params, now(), ttl)
```
The read-modify-write is atomic because the **whole decision runs as one script on the
store** — no two instances interleave. Cost: each algorithm needs a matching script, so a
custom algorithm must supply one (the pure-function port weakens for this path).

## Option B — optimistic CAS (keeps the pure RateLimitAlgorithm)

```
class CasCounterStore implements CounterStore:
    atomicApply(key, ttl, fn):
        repeat up to N times:
            (state, version) = store.readVersioned(key)
            outcome = fn(state)                     // the pure decide() runs client-side
            if store.compareAndSet(key, version, outcome.nextState, ttl):
                return outcome
            // another instance won the race — retry with fresh state
        fail StoreContention                        // surfaces to the engine's fail-mode
```
Keeps `decide` pure and pluggable (the extension seam intact); costs extra round-trips
under contention. Good default; move only hot keys to Option A.

## Option C — approximate local counter (scale over exactness)

```
// each instance owns a slice of the global budget, refilled by a periodic sync
localBudget = globalLimit / instanceCount

function checkApprox(policy, key):
    if localBucket(key).tryConsume():
        return ALLOW
    return DENY
// overshoots the global limit by up to (instanceCount - 1) — accepted when approximate is fine
// reconcile / rebalance budgets periodically via a shared pool or gossip
```

## Fail-mode (unchanged engine wrapper, now load-bearing)

```
try:
    outcome = store.atomicApply(key, ttl, fn)
catch StoreUnavailable or StoreContention:
    return failModeDecision(policy)                 // open -> ALLOW, closed -> DENY (per policy)
```

Notes:
- **Central store** = exact-ish + one hop per check + the store is the **bottleneck /
  SPOF** → run it HA and **shard by key**.
- **CAS** preserves the pure algorithm; the **script** duplicates the logic into the store.
- **Approximate** trades exactness for no per-check hop.
- **Through-line:** atomicity moved from an **app-side lock** (L1) to a **store-side script
  / CAS** (L3); the counter that was an **in-process map** is now a **shared store**; under
  CAS the `RateLimitAlgorithm` stays the same pure code written at Level 1.
