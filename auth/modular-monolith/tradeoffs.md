# Auth — Level 1: Trade-offs

## What's good (why start here)
- **Simplicity:** a call is a function call — no network, serialization, retries, or
  partial failure.
- **Atomicity:** one transaction spans account write + credential + audit → easy
  consistency.
- **Cheap identity:** "who is this user" is a local DB lookup, synchronous.
- **One of everything:** one codebase, one deploy, one CI, one place to debug.
- **Easy refactor:** module boundaries are a facade + discipline; you can move them
  without renegotiating a network contract.
- **Fast:** no inter-service latency; p95 is trivially met.

## What hurts (the limits)
- **Scales as a block:** you cannot give Auth more machines without scaling everything.
- **Shared session state couples instances:** it must live in a shared store (DB/cache),
  not process memory, or horizontal scaling breaks.
- **No fault isolation:** a bug in another module can take the whole app down.
- **Shared DB contention** grows as modules multiply.
- **Boundary erosion risk:** nothing *physically* stops another module from reaching
  into Auth's tables — only the facade + code review do.
- **Single deploy cadence:** shipping Auth means redeploying the whole app.

## Password ops at Level 1
- **Revocation is a local delete:** reset / change call `revokeAllFor` — one `UPDATE` over
  the account's session/token rows, instantly effective. Enjoy it; Level 3 makes this the
  hard part.
- **The out-of-band send is the one non-transactional step:** the token row and the
  email/SMS can't share a transaction. Commit the row **then** send — a failed send just
  means no email (the user retries), never a live token without a row. (A real
  outbox/queue makes this reliable; noted, not built.)
- **Reset token reuses the bearer-token pattern:** high-entropy secret, stored as a fast
  hash, shown once — no new crypto, just a new table.

## When Level 1 is the right choice
- Early product, one team, modest load. Most systems should **start here and stay
  longer than they expect** — the limits above are not yet real costs.

## What would push you up a level
- **→ Level 2 (module):** you want the boundary enforced by the package/compiler, not
  just discipline — a swappable, independently-testable Auth package with zero leakage.
- **→ Level 3 (microservice):** a real forcing function — Auth must scale or deploy
  **independently**, needs **fault isolation**, or a **separate team** owns it. Only
  then does the cost of network calls, distributed auth, and data ownership pay off.

## Carry-forward to the next level
- The facade (`identify`, `login`, …) is the **seam**: it becomes a package API at
  Level 2, then a **network contract** at Level 3.
- **Shared session state** is the first thing the network forces you to rethink —
  stateful session lookup gives way to a stateless token.
- **Reset / change must revoke every credential** — trivial here (a DB delete), but it is
  exactly what a **stateless token** can't do on demand at Level 3 without a denylist or
  short TTL + refresh-token revocation. This is the canary for the L3 design.
