# Rate-Limiting — Glossary

Shared terms (ubiquitous language) for the Rate-Limiting component. One definition each.

| Term | Meaning |
|---|---|
| **Policy** | A named rule: algorithm + params + key dimension + fail-mode. Lives in config. |
| **Key** | A `(policy, dimension value)` pair — the concrete subject of a check, e.g. `(login-per-ip, 1.2.3.4)`. |
| **Dimension** | What a policy keys on: IP, identity, route, API key, tenant. |
| **Decision** | The result of a check: `allowed, remaining, limit, retryAfter, resetAt`. |
| **RateLimitAlgorithm** | The pluggable strategy that decides allow/deny + next state. Built-ins + client-extensible. |
| **Token bucket** | Capacity N, refills R/sec; a request spends 1 token; empty = deny. Allows bursts up to N. |
| **Sliding window counter** | Weighted count over the trailing window using previous + current fixed-window counts. Smooth, O(1) state. |
| **Fixed window** | Count per discrete window, reset at the boundary. Simplest; allows a 2x edge burst. |
| **CounterStore** | The atomic per-key state store (in-process / Redis / distributed). Owns atomicity + TTL. |
| **Atomic check** | A read-modify-write applied as one indivisible step so concurrent checks cannot double-admit. |
| **Burst** | A short spike above the steady rate, tolerated up to the bucket capacity. |
| **Refill rate** | How fast a token bucket regains tokens (e.g. 1/sec). |
| **Window** | The time interval a window algorithm counts over (e.g. 1 minute). |
| **Fail-open** | On store failure, **allow** (favor availability). |
| **Fail-closed** | On store failure, **deny** (favor protection). |
| **Retry-After** | HTTP header telling the client when to retry after a 429. |
| **RateLimit-* headers** | `Limit / Remaining / Reset` headers exposing the budget to clients. |
| **Clear-on-success** | Resetting a key after a legitimate success so the user is not penalized (e.g. Auth after login). |
| **Clock** | Injected time source; makes refill / window math testable. |
