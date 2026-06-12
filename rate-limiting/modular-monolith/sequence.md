# Rate-Limiting — Level 1: Sequences

Token bucket shown in full; the other algorithms plug into the same `decide` step.
**Every arrow is an in-process call.**

## Check (token bucket, in-process atomic)

```mermaid
---
title: Rate-Limiting — Level 1 Check (token bucket, in-process atomic)
---
sequenceDiagram
    autonumber
    participant Co as Consumer (Auth / middleware)
    participant E as RateLimiter
    participant P as PolicyRegistry
    participant AR as AlgorithmRegistry
    participant S as CounterStore (in-process)
    participant Cl as Clock

    Co->>E: check("login-per-identity", identifier)
    E->>P: get("login-per-identity")
    E->>AR: get("token_bucket")
    E->>S: atomicApply(key, ttl, fn)
    activate S
    Note over S: acquire per-key lock (atomicity)
    S->>Cl: now()
    Note over S: TokenBucket.decide —<br/>refill = min(cap, tokens + elapsed*rate)
    alt tokens >= 1
        S->>S: store(tokens - 1), build ALLOW
    else empty
        S->>S: keep state, build DENY (retryAfter)
    end
    Note over S: release lock
    deactivate S
    E-->>Co: Decision
    alt denied
        Co-->>Co: respond 429 + Retry-After
    end
```

## Compose — Auth's two buckets (delta)
- Auth ANDs two checks: `check("login-per-identity", id)` and `check("login-per-ip", ip)`.
- **Denied if either denies**; the surfaced `retryAfter` is the **larger** of the two.
- This is consumer-side composition — the engine has no special "two-bucket" mode.

## Clear on success (delta)
- After a successful login, Auth calls `clear("login-per-identity", id)` (and the IP key)
  so a legitimate user is not left throttled by earlier failures.
- `clear` removes the key's state — the next `check` starts from a full bucket.
