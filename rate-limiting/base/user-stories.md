# Rate-Limiting — User Stories & Requirements

Functional stories (the "user" is usually another component) plus non-functional
requirements, shared by all three levels. The concrete store, atomicity mechanism, and
deployment are each level's job — see the level docs.

**Conventions:** story = *As a … I want … so that …*; acceptance criteria = *Given /
When / Then*. "Consumer" = a component or middleware that calls the limiter (e.g. Auth).

---

## Functional stories

### US-1 — Check a limit (consume)
*As a consumer, I want to check a key against a named policy, so that I can allow or
reject the action and tell the caller when to retry.*

- **Given** a key **under** its policy's limit, **when** I `check`, **then** it returns
  **allowed** with `remaining` and `resetAt`, and the hit is **counted atomically**.
- **Given** a key **at or over** the limit, **when** I `check`, **then** it returns
  **denied** with `retryAfter`, and no further budget is consumed.
- **Given** **two concurrent** checks at the boundary, **when** both run, **then** **at
  most the limit** is admitted — no double-admit race (atomic).

### US-2 — Compose policies (AND)
*As a consumer, I want to AND several policy checks (e.g. per-identity and per-IP), so
that I can throttle on more than one dimension at once.*

- **Given** two policies, **when** **either** denies, **then** the overall result is
  **denied** and `retryAfter` is the **longer** of the two.
- **Given** both allow, **then** the overall result is **allowed**.

### US-3 — Clear on success
*As a consumer (e.g. Auth after a successful login), I want to reset a key's counter, so
that a legitimate success does not leave the user penalized.*

- **Given** a key, **when** I `clear` it, **then** its state is removed and the next
  `check` starts fresh.

### US-4 — Define a policy (config)
*As an operator, I want to declare named policies in config (algorithm + params + key
dimension + fail-mode), so that limits change without code edits.*

- **Given** a new policy in config, **when** the app loads, **then** `check(policy, key)`
  uses it with **no code change** (OCP).
- **Given** a policy names an unknown algorithm, **when** the app loads, **then** it
  **fails loud at startup** — it never silently allows everything.

### US-5 — Plug in an algorithm
*As a developer, I want to implement `RateLimitAlgorithm` and register it, so that I can
add a custom algorithm without modifying the engine.*

- **Given** a custom algorithm registered as `"x"`, **when** a policy names `"x"`,
  **then** the engine uses it — the engine code is untouched.

### US-6 — Read without consuming (peek / headers)
*As a consumer, I want to read `remaining` / `resetAt` without consuming budget, so that
I can emit `RateLimit-*` headers or pre-flight a decision.*

- **Given** a key, **when** I `peek`, **then** it returns the current view **without
  mutating state**.

---

## Non-functional requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-1 | Correctness | `check` is **atomic** — never admits more than the limit under concurrency. |
| NFR-2 | Performance | Minimal hot-path overhead — **one** store round-trip per check; small p99 cost. |
| NFR-3 | Availability | On store failure, behaviour is **per-policy fail-open or fail-closed** — explicit, never undefined. |
| NFR-4 | Accuracy | `retryAfter` / `resetAt` reflect the algorithm's true next-allow time. |
| NFR-5 | Config | Policies (algorithm, params, key, fail-mode) from **config**, reloadable, not hardcoded. |
| NFR-6 | Extensibility | New algorithm = implement the interface + register; **no engine change** (OCP). |
| NFR-7 | Interop | Deny maps to **HTTP 429 + `Retry-After`**; allow may expose `RateLimit-Limit/Remaining/Reset`. |
| NFR-8 | Isolation | Each `(policy, key)` state is isolated — one key cannot affect another. |
| NFR-9 | Bounded state | Per-key state is **small** and **expires (TTL)** so abandoned keys do not accumulate. |
| NFR-10 | Observability | Allowed / denied counts emitted per policy. |
| NFR-11 | Testability | Time comes from an **injected clock** — no hidden global `now()`. |
| NFR-12 | Security | Key dimension values (IP, identifier) are **normalized + length-bounded** before use, to prevent key collision / blow-up. |

---

## References (owned elsewhere)

- **CounterStore** — the atomic state store (in-process map / Redis / distributed) is
  chosen per level; the component depends on the **port**, not a vendor.
- **Clock**, **Config**, **Metrics** — injected infra.
- **Consumers** (e.g. Auth) own how they **derive a key** from a request (which IP /
  identifier) and how they translate a deny into an HTTP response.

## Deferred (out of base scope)

- Usage / billing quotas (long-horizon accounting).
- Concurrency (in-flight) limiting — a semaphore, not a rate.
- Distributed quota reconciliation across regions.
- A dynamic policy-config service / admin UI.
