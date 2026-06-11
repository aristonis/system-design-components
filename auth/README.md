# Auth — Component Design

Authentication: **register / login / logout**. Web-only, two client channels
(browser → session, API → bearer token). Designed once in `base/`, then realized at
**three levels of decoupling** (modular-monolith → module → microservice).

> Design only — docs, UML (Mermaid), and language-agnostic pseudocode. No runnable code.

## Scope

- **In:** register, login, logout.
- **Out / separate components:** authorization (roles/permissions), rate-limiting (infra).
- **Deferred:** reset password, email verification, 2FA, social login.

## How to read it

```
base/   →   modular-monolith/   →   module/   →   microservice/
(WHAT,        (L1: in-process)       (L2: ports    (L3: own service,
 level-                              & adapters)    stateless JWT)
 independent)
```

The **`base/` requirements never change** — each level only changes *how the component
is split and deployed*. Read `base/` first, then the levels in order.

## base/ — the level-independent contract

| File | What it holds |
|---|---|
| [context.md](base/context.md) | environment, channels, the single-flow / per-channel auth model |
| [user-stories.md](base/user-stories.md) | register / login / logout stories + non-functional requirements |
| [use-case.md](base/use-case.md) | use-case diagram |
| [crc-cards.md](base/crc-cards.md) | collaboration diagram + CRC cards |
| [domain-model.md](base/domain-model.md) | class diagram (`Authenticator`, `CredentialIssuer`, `Identifier`, …) |
| [data-model.md](base/data-model.md) | logical ER; single `ACCOUNT`; identifier backings A/B |
| [glossary.md](base/glossary.md) | ubiquitous language |

## The three levels

Each level has the same four files: `architecture` · `sequence` · `pseudocode` · `tradeoffs`.

| Level | Boundary | Calls | Deploy / DB | "Who is this?" | Scale Auth alone |
|---|---|---|---|---|---|
| [1 — modular-monolith](modular-monolith/architecture.md) | discipline + facade | in-process | one app, shared DB | local DB lookup | no |
| [2 — module](module/architecture.md) | compiler / package | in-process via ports | one app, shared DB | local port call | no |
| [3 — microservice](microservice/architecture.md) | network contract | network | own service, own DB | local JWT verify (stateless) | **yes** |

Full comparison: [microservice/tradeoffs.md](microservice/tradeoffs.md).

## Key design decisions

- **One login flow**, channel selects a **credential-issuance strategy** (session/token) from a registry — adding a channel is an insert, not an edit (OCP).
- **Generalized identifier:** `findByIdentifier(identifier)` (email / username / NID); storage is the implementer's choice — inline (A) or `IDENTIFIER` table (B).
- **Single `ACCOUNT`** — "admin" is a role owned by the separate Authorization component, not an Auth concern.
- **Token mechanism evolves per level:** opaque server-stored token (L1/L2) → stateless JWT + refresh token (L3). This is the central monolith→microservice lesson.
- **Cross-cutting deps referenced, not owned:** rate-limiting is its own infra component.

## Status

Auth design complete and internally consistent across `base/` + all three levels.
