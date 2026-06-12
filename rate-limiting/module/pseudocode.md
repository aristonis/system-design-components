# Rate-Limiting — Level 2: Pseudocode

The **engine and algorithm bodies are unchanged** from Level 1
(`modular-monolith/pseudocode.md`). What Level 2 adds is **explicit ports** and a
**composition root** that injects adapters — including a client's own algorithm.

## Driving port (the package's public contract)

```
interface RateLimitApi:
    check(policyName, key) -> Decision
    peek(policyName, key)  -> Decision
    clear(policyName, key) -> void
```

## Driven ports (what the core needs from outside)

```
interface CounterStore:        atomicApply(key, ttl, fn) -> Outcome ;  read(key) -> State | null ;  clear(key)
interface Clock:               now() -> Time
interface RateLimitAlgorithm:  decide(state, policy, now) -> Outcome ;  project(state, policy, now) -> Decision
```
`RateLimitAlgorithm` is a **driven port the client may implement** — the extension seam.

## Core (implements RateLimitApi, depends only on ports)

```
class RateLimiter implements RateLimitApi:
    constructor(policies, algorithms, store, clock): ...
    // check / peek / clear bodies are exactly Level 1's pseudocode,
    // but policies, algorithms, store, and clock are all injected PORTS.
```

## Composition root (host wires adapters → ports)

```
function buildRateLimiter(config, redis):
    algorithms = registry()
    algorithms.register("token_bucket",   TokenBucket())
    algorithms.register("fixed_window",   FixedWindow())
    algorithms.register("sliding_window", SlidingWindowCounter())
    // client extension — no engine change:
    // algorithms.register("my_algo", MyAlgorithm())

    policies = PolicyRegistry(config.policies)               // fail loud on unknown algorithm
    store    = config.useRedis ? RedisCounterStore(redis) : InMemoryCounterStore()
    return RateLimiter(policies, algorithms, store, SystemClock())
```

## A custom algorithm is just the port + a registration

```
class MyAlgorithm implements RateLimitAlgorithm:
    decide(state, policy, now):  ...                         // pure logic over state
    project(state, policy, now): ...                         // peek view

// register it, name it in a policy — done. The engine never changes (OCP).
```

## Deterministic algorithm test (fake clock, in-memory store)

```
function test_token_bucket_refills_over_time():
    clock = FakeClock(at = t0)
    rl = RateLimiter(
        policies   = PolicyRegistry({ "api": Policy(algorithm = "token_bucket",
                                                    params = { capacity: 2, refillPerSec: 1 }) }),
        algorithms = registry().register("token_bucket", TokenBucket()),
        store      = InMemoryCounterStore(),
        clock      = clock)

    assert rl.check("api", "k").allowed == true      // 2 -> 1
    assert rl.check("api", "k").allowed == true      // 1 -> 0
    assert rl.check("api", "k").allowed == false     // empty -> DENY
    clock.advance(1 second)
    assert rl.check("api", "k").allowed == true      // refilled 1 -> 0
```

Swapping `InMemoryCounterStore` for `RedisCounterStore` later changes **one line in the
composition root**; the engine, algorithms, policies, and tests are untouched (OCP / DIP).
