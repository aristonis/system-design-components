# Auth — Level 2: Trade-offs

## What's good
- **Boundary enforced by the compiler**, not by discipline — internals and tables
  cannot leak; callers must use `AuthApi`.
- **Independently testable** — the core runs against fake adapters with no DB, so tests
  are fast and focused.
- **Swappable infrastructure** — new DB, hasher, cache, or rate-limiter = a new adapter;
  the core is untouched (OCP / DIP).
- **Reusable** — the package drops into another host app by supplying adapters.
- **Clear contract** — `AuthApi` is explicit, documented, and stable.

## What it costs
- **More ceremony** — ports, adapters, DTOs, and a composition root are extra moving
  parts; over-applied to a tiny app it is overkill (YAGNI).
- **Indirection** — one more hop to read when tracing a call.
- **No deployment win** — runtime is still one process; **scalability is identical to
  Level 1**. You did not buy independent scaling, only a clean boundary.
- **Wiring complexity** — the composition root must assemble everything correctly.

## When Level 2 is the right choice
- The module is shared across apps, or several teams touch it, or you want to lock the
  boundary before it erodes — but you do **not** yet need independent deployment/scaling.
- Often the **best long-term home** for a component: most code never needs Level 3.

## What would push you to Level 3
- A real forcing function: Auth must **scale or deploy independently**, needs **fault
  isolation**, or a **separate team/runtime** owns it. Only then is the network cost
  worth paying.

## Carry-forward to Level 3
- `AuthApi` (the driving port) becomes the **network contract** (REST/gRPC).
- The driven ports are the seams that may move across the network — and the
  `CredentialStore`/session lookup is the one that pushes you toward **stateless tokens**
  so downstream services need not call Auth on every request.
