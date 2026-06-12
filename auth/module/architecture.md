# Auth — Level 2: Module (Ports & Adapters)

**Level 2 = the Auth module extracted into a self-contained package**, with its
boundary enforced by the **package/compiler — not discipline**. Runtime is unchanged
from Level 1 (still one app, one process, in-process calls); what changes is the
**structure**: the core depends only on **ports** (interfaces), and **adapters** are
plugged in from outside (hexagonal / ports-and-adapters).

## Shape

```mermaid
---
title: Auth — Level 2 Module (ports & adapters)
---
flowchart LR
    subgraph Outside["Host application — composition root wires adapters"]
      direction TB
      WebAd["Web controller adapter"]
      ApiAd["API controller adapter"]
      DBAd["AccountRepository adapter (DB)"]
      StoreAd["CredentialStore adapter (DB)"]
      HashAd["PasswordHasher adapter (KDF)"]
      RLAd["RateLimiter adapter"]
      AuditAd["AuditLog adapter"]
      RTSAd["ResetTokenStore adapter (DB)"]
      NotifAd["Notifier adapter (email / SMS)"]
    end

    subgraph Pkg["Auth package — sealed"]
      direction TB
      Drive{{"AuthApi — driving port<br/>register / login / logout / identify"}}
      PwdDrive{{"PasswordApi — driving port<br/>forgot / reset / change"}}
      Core["Core: Authenticator + PasswordManager + domain<br/>(package-private, hidden)"]
      RepoP{{"AccountRepository port"}}
      StoreP{{"CredentialStore port"}}
      HashP{{"PasswordHasher port"}}
      RLP{{"RateLimiter port"}}
      AuditP{{"AuditLog port"}}
      RTSP{{"ResetTokenStore port"}}
      NotifP{{"Notifier port"}}
    end

    WebAd --> Drive
    ApiAd --> Drive
    WebAd --> PwdDrive
    ApiAd --> PwdDrive
    Drive --> Core
    PwdDrive --> Core
    Core --> RepoP
    Core --> StoreP
    Core --> HashP
    Core --> RLP
    Core --> AuditP
    Core --> RTSP
    Core --> NotifP
    RepoP -. implemented by .-> DBAd
    StoreP -. implemented by .-> StoreAd
    HashP -. implemented by .-> HashAd
    RLP -. implemented by .-> RLAd
    AuditP -. implemented by .-> AuditAd
    RTSP -. implemented by .-> RTSAd
    NotifP -. implemented by .-> NotifAd
```

## Ports

- **Driving ports** (`AuthApi`, `PasswordApi`) — the only ways *in*. `AuthApi` fronts the
  `Authenticator`; `PasswordApi` fronts the `PasswordManager`. Controllers (adapters) call
  them; they cannot reach the core any other way.
- **Driven ports** (`AccountRepository`, `CredentialStore`, `PasswordHasher`,
  `RateLimiter`, `AuditLog`, plus `ResetTokenStore` and `Notifier`) — the only ways *out*.
  The core declares what it needs; the host supplies implementations.
- The core depends on **ports only, never on a concrete adapter** (Dependency Inversion).

## What is hidden vs exported

| Exported (public) | Hidden (package-private) |
|---|---|
| `AuthApi`, `PasswordApi`, the port interfaces, request/response DTOs, error types | `Authenticator` / `PasswordManager` implementations, domain entities' internals, issuer registry, hashing + token-hashing details |

Nothing outside the package can construct or touch the core directly — the compiler
enforces it.

## Composition root (who wires it)

The **host application** assembles the adapters and injects them — the package never
names a concrete implementation:

```
auth = Authenticator(
    accounts      = DbAccountRepository(db),
    credentials   = DbCredentialStore(db),
    hasher        = Argon2Hasher(),
    rateLimiter   = SharedCacheRateLimiter(cache),
    audit         = DbAuditLog(db),
    issuers       = { web: SessionIssuer, api: TokenIssuer },
)

passwords = PasswordManager(
    accounts      = DbAccountRepository(db),
    credentials   = DbCredentialStore(db),     // for revokeAllFor
    hasher        = Argon2Hasher(),
    resetTokens   = DbResetTokenStore(db),
    notifier      = EmailNotifier(mailer),     // out-of-band adapter
    rateLimiter   = SharedCacheRateLimiter(cache),
    audit         = DbAuditLog(db),
)
```

## What does NOT change

- **Deployment topology:** still one application, one process, one DB — adapters can use
  the same shared database as Level 1.
- **Therefore the scalability profile is the same as Level 1** — this is a
  *code-structure* level, not a *deployment* level. You do not get independent scaling
  here; you get a clean, swappable, leak-proof boundary.

## What you gain

- **Enforced boundary** — no other module can reach Auth's internals or tables; they
  must go through `AuthApi`.
- **Independent testability** — unit-test the core with **fake adapters**, no DB needed.
  Even the out-of-band recovery flow is testable: a **fake `Notifier`** captures the raw
  token in memory, so a test can request a reset and then consume that token — no mail
  server.
- **Swappability** — change DB, hasher, or rate-limiter by writing a new adapter; the
  core is untouched (OCP).
- **Reusability** — drop the package into a different host app by supplying adapters.

## The Level-2 lesson (sets up Level 3)

`AuthApi` is now a **real, explicit contract** instead of an informal facade. That
contract is exactly what becomes a **network API** at Level 3 — and the driven ports
(`CredentialStore`, etc.) are exactly the seams where, at Level 3, an in-process call
becomes a call across the network.

The `Notifier` port is a preview of Level 3's asynchrony: out-of-band delivery is the
one collaborator that is already naturally *fire-and-forget*, so it becomes a queue /
message publish at Level 3 with **no change to the core** — only a new adapter.
