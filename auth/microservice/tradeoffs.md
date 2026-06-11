# Auth — Level 3: Trade-offs

## What's good (why pay for this)
- **Independent scale** — give auth-service its own machines; identity validation has
  **no central bottleneck** (every service verifies tokens locally).
- **Independent deploy** — ship Auth without redeploying anything else.
- **Fault isolation** — an Auth crash does not take down order/wallet; and existing
  valid tokens keep working through a short Auth outage (graceful degradation).
- **Separate ownership** — a dedicated team/runtime can own Auth end to end.

## What it costs
- **Network everywhere** — login/refresh can time out, be slow, or fail partially;
  retries, timeouts, idempotency are now your problem.
- **Distributed complexity** — gateway, service discovery, tracing, per-service
  monitoring, more deploy pipelines.
- **Revocation is eventual** — stateless JWTs cannot be revoked instantly; you rely on
  short TTLs + refresh revocation + an optional denylist.
- **Key management** — signing keys, rotation, JWKS distribution must be operated.
- **Eventual consistency** — if other services cache user data, it lags account changes.
- **Harder local dev/testing** — multiple services to run; integration tests get heavier.
- **Web sessions need a BFF** — cookie-based browser flows must be bridged to tokens.

## When Level 3 is justified
- Only with a **real forcing function**: Auth must scale or deploy independently, needs
  fault isolation, or a separate team owns it. Without one, you are paying all the costs
  above for none of the benefits — stay at Level 1 or 2.

## What you lose vs the monolith
- Simple **atomic transactions** spanning Auth + other modules.
- **Instant revocation** and a single, local "who is this" lookup.
- **One place to debug.**

---

## The three levels at a glance (the journey)

| Aspect | L1 Modular Monolith | L2 Module (ports & adapters) | L3 Microservice |
|---|---|---|---|
| Boundary enforced by | discipline + facade | compiler / package | network contract |
| Call type | in-process | in-process via ports | network |
| Deployment | one app | one app | own service |
| Database | shared | shared | own (private) |
| "Who is this?" | local DB lookup | local port call | local JWT verify (stateless) |
| Scale Auth alone | no | no | **yes** |
| Revocation | instant | instant | eventual (short TTL) |
| Failure surface | in-process | in-process | network + partial failure |
| Complexity | low | medium | high |
| Best for | most products | shared/long-lived modules | a real forcing function |

**The lesson:** each level only earns its complexity when a real constraint demands it.
The same Auth requirements (`base/`) were satisfied at all three — what changed was
*how the component is split and deployed*, and every split bought capability at the
price of complexity. Move up the ladder only when something forces you to.
