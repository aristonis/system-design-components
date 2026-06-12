# Rate-Limiting — Domain Model

Class diagram (level-independent). It fixes the types and the **extension seam**; wiring
and the concrete store are each level's concern.

```mermaid
---
title: Rate-Limiting — Domain Model (class diagram)
---
classDiagram
    class RateLimiter {
      +check(policyName, key) Decision
      +peek(policyName, key) Decision
      +clear(policyName, key) void
    }
    class Policy {
      +name : String
      +algorithm : String
      +params : Map
      +keyDimension : String
      +failMode : FailMode
    }
    class Decision {
      +allowed : bool
      +remaining : int
      +limit : int
      +retryAfter : Duration
      +resetAt : Time
    }
    class RateLimitAlgorithm {
      <<interface>>
      +decide(state, policy, now) Outcome
    }
    class Outcome {
      +decision : Decision
      +nextState : State
    }
    class State {
      <<abstract>>
    }
    class TokenBucketState {
      +tokens : float
      +lastRefillAt : Time
    }
    class WindowState {
      +count : int
      +windowStart : Time
    }
    class SlidingState {
      +prevCount : int
      +currCount : int
      +currWindowStart : Time
    }
    class TokenBucket
    class FixedWindow
    class SlidingWindowCounter
    class CounterStore {
      <<interface>>
      +atomicApply(key, fn, ttl) Outcome
      +clear(key) void
    }
    class Clock {
      <<interface>>
      +now() Time
    }
    class AlgorithmRegistry {
      <<interface>>
      +register(name, algorithm) void
      +get(name) RateLimitAlgorithm
    }
    class PolicyRegistry {
      <<interface>>
      +get(name) Policy
    }

    RateLimiter --> PolicyRegistry
    RateLimiter --> AlgorithmRegistry
    RateLimiter --> CounterStore
    RateLimiter --> Clock
    RateLimiter ..> Decision : creates
    RateLimitAlgorithm <|.. TokenBucket
    RateLimitAlgorithm <|.. FixedWindow
    RateLimitAlgorithm <|.. SlidingWindowCounter
    RateLimitAlgorithm ..> Outcome : returns
    State <|-- TokenBucketState
    State <|-- WindowState
    State <|-- SlidingState
    TokenBucket ..> TokenBucketState : uses
    FixedWindow ..> WindowState : uses
    SlidingWindowCounter ..> SlidingState : uses
    AlgorithmRegistry o-- RateLimitAlgorithm
```

Design notes:
- **Strategy + registry (the extension point):** `RateLimitAlgorithm` is selected by name
  from `AlgorithmRegistry`. A client adds an algorithm by **implementing the interface and
  registering it** — `RateLimiter` never changes (OCP), exactly like Auth's
  `CredentialIssuer` registry. The three built-ins ship in the box.
- **Algorithm is pure:** `decide(state, policy, now)` does **no I/O and reads no clock** —
  `now` is passed in. Each algorithm is trivially unit-testable, and atomicity stays out
  of it.
- **Atomicity lives in `CounterStore.atomicApply`:** the engine hands the store a function
  `fn(state) -> Outcome`; the store applies it **atomically** (store-side script or
  compare-and-set retry) so concurrent checks cannot double-admit. *How* is each level's
  job. (The algorithm's `decide` is what `fn` calls.)
- **State shapes differ per algorithm** but are all tiny (2–3 fields) and carry a TTL.
- `Clock` is an interface so refill / window timing is deterministic in tests.
