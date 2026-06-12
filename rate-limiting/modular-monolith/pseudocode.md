# Rate-Limiting — Level 1: Pseudocode

Language-agnostic. Level-1 wiring: collaborators are **in-process**; atomicity is a
**per-key lock** (single instance) — swap the store for a shared cache to go
multi-instance. The three built-in algorithms are defined here once (they are pure and
level-independent); higher levels reuse them and only change the store.

## Registries (resolve by name — fail loud)

```
algorithms = {
  "token_bucket":   TokenBucket,
  "fixed_window":   FixedWindow,
  "sliding_window": SlidingWindowCounter,
}
function algorithmFor(name):
    a = algorithms[name]
    if a is null: fail "unknown algorithm: " + name      // startup / no silent allow
    return a

policies = loadFromConfig()                              // name -> Policy
function policyFor(name):
    p = policies[name]
    if p is null: fail "unknown policy: " + name
    return p
```
A client adds an algorithm by `algorithms["my_algo"] = MyAlgorithm` + a policy that names
it — the engine below never changes (OCP).

## Engine

```
function check(policyName, rawKey):
    policy = policyFor(policyName)
    algo   = algorithmFor(policy.algorithm)
    key    = "rl:" + policyName + ":" + normalize(rawKey)
    try:
        outcome = store.atomicApply(key, ttlFor(policy), fn(state):
                      return algo.decide(state, policy, clock.now()))
    catch StoreUnavailable:
        metrics.inc(policyName, "store_error")
        return failModeDecision(policy)                  // open -> ALLOW, closed -> DENY
    metrics.inc(policyName, outcome.decision.allowed ? "allowed" : "denied")
    return outcome.decision

function peek(policyName, rawKey):                        // no consume
    policy = policyFor(policyName) ; algo = algorithmFor(policy.algorithm)
    key    = "rl:" + policyName + ":" + normalize(rawKey)
    state  = store.read(key)
    return algo.project(state, policy, clock.now())       // remaining/resetAt, no write

function clear(policyName, rawKey):
    store.clear("rl:" + policyName + ":" + normalize(rawKey))
```

## Compose (consumer-side helper — e.g. Auth)

```
function checkAll(checks):                                // checks = [(policy, key), ...]
    decisions  = [ check(p, k) for (p, k) in checks ]
    allowed    = every d in decisions: d.allowed
    retryAfter = max(d.retryAfter for d in decisions where not d.allowed) or 0
    return Decision(allowed, retryAfter = retryAfter, remaining = min(d.remaining ...))
```

## TokenBucket (pure decision — the worked example)

```
class TokenBucket implements RateLimitAlgorithm:
  decide(state, policy, now):
    cap  = policy.params.capacity
    rate = policy.params.refillPerSec
    s    = state ?? TokenBucketState(tokens = cap, lastRefillAt = now)

    elapsed = max(0, now - s.lastRefillAt)
    tokens  = min(cap, s.tokens + elapsed * rate)         // refill by elapsed time

    if tokens >= 1:
        next = TokenBucketState(tokens = tokens - 1, lastRefillAt = now)
        full = (cap - next.tokens) / rate                 // time back to full
        return Outcome(Decision(true,  floor(next.tokens), cap, 0,    now + full), next)
    else:
        wait = (1 - tokens) / rate                        // time until 1 token
        next = TokenBucketState(tokens = tokens, lastRefillAt = now)
        return Outcome(Decision(false, 0,                cap, wait, now + wait), next)
```

## FixedWindow (pure)

```
class FixedWindow implements RateLimitAlgorithm:
  decide(state, policy, now):
    limit = policy.params.limit
    win   = policy.params.windowSec
    start = floorToWindow(now, win)
    s     = (state and state.windowStart == start) ? state : WindowState(0, start)

    if s.count < limit:
        next = WindowState(s.count + 1, start)
        return Outcome(Decision(true,  limit - next.count, limit, 0,             start + win), next)
    else:
        return Outcome(Decision(false, 0,                 limit, start + win - now, start + win), s)
```
Simplest, but two full windows can fire back-to-back across the boundary → up to **2x**
the limit in a short span (the edge-burst flaw).

## SlidingWindowCounter (pure)

```
class SlidingWindowCounter implements RateLimitAlgorithm:
  decide(state, policy, now):
    limit = policy.params.limit
    win   = policy.params.windowSec
    start = floorToWindow(now, win)
    s     = roll(state, start, win)                       // shift curr -> prev as the window advances
    overlap  = (win - (now - start)) / win                // fraction of previous window still in view
    estimate = s.prevCount * overlap + s.currCount

    if estimate < limit:
        next = withCurr(s, s.currCount + 1)
        return Outcome(Decision(true,  floor(limit - estimate - 1), limit, 0, start + win), next)
    else:
        return Outcome(Decision(false, 0, limit, approxRetry(s, policy, now), start + win), s)
```
Smooths the fixed-window edge burst with O(1) state; `approxRetry` is the estimated time
until the weighted `estimate` drops below `limit`.

## In-process CounterStore (Level 1)

```
class InProcessCounterStore implements CounterStore:
  map   = concurrentMap()                                 // key -> { state, expiresAt }
  locks = stripedLocks()

  atomicApply(key, ttl, fn):
    with locks.for(key):                                  // atomicity = per-key lock (one process)
        entry = map.get(key)
        state = (entry and entry.expiresAt > now()) ? entry.state : null
        outcome = fn(state)                               // runs algorithm.decide
        map.put(key, { state: outcome.nextState, expiresAt: now() + ttl })
        return outcome

  read(key):  entry = map.get(key) ; return (entry and entry.expiresAt > now()) ? entry.state : null
  clear(key): map.remove(key)
  // a periodic sweep (or lazy eviction on access) drops expired entries — bounded memory
```

**The one swap that changes everything:** replace `InProcessCounterStore` with a
shared-cache store whose `atomicApply` is a **server-side script** — now there is no
app-side lock and the counter is correct **across instances**. The engine, algorithms,
and policies are untouched.

Config: per-policy `params` (`capacity` + `refillPerSec`, or `limit` + `windowSec`),
`failMode`, `ttl`; `algorithm` name; `clock` injected.
