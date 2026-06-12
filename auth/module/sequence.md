# Auth — Level 2: Sequences

Same login logic as Level 1, but now every crossing goes **through a port**. Still
**in-process** — the value is the enforced boundary, not new runtime behaviour.

```mermaid
---
title: Auth — Level 2 Login (through ports & adapters)
---
sequenceDiagram
    autonumber
    actor C as Client
    participant Ad as Controller adapter
    participant Api as AuthApi (driving port)
    participant Core as Authenticator (core)
    participant RLp as RateLimiter port→adapter
    participant Rp as AccountRepository port→adapter
    participant Hp as PasswordHasher port→adapter
    participant Sp as CredentialStore port→adapter
    participant Lp as AuditLog port→adapter

    C->>Ad: POST /login {identifier, password} (channel)
    Ad->>Api: login(identifier, password, channel)
    Api->>Core: (same call, boundary enforced)
    Core->>RLp: check(identifier, ip)
    alt blocked
        Core->>Lp: record(login_blocked)
        Core-->>Ad: Error(429)
    else allowed
        Core->>Rp: findByIdentifier(identifier)
        Core->>Hp: verify(password, hash)   (timing-safe)
        alt invalid
            Core->>Lp: record(login_failure)
            Core-->>Ad: Error(401 generic)
        else valid
            Core->>Sp: save(credential via issuerFor(channel))
            Core->>Lp: record(login_success)
            Core-->>Ad: Credential
            Ad-->>C: Set-Cookie (web) / 200 {token} (api)
        end
    end
    Note over Api,Sp: every arrow crosses a PORT — adapters are injected by the host.<br/>Still in-process — no network.
```

Note: the **branch logic is identical to Level 1** (`modular-monolith/sequence.md`).
The only difference is that the core reaches its collaborators **exclusively through
ports**, so each one can be replaced or faked without touching the core.

Register and logout follow the same port-mediated shape.

## Password operations (delta)
- `forgot / reset / change` enter through a second driving port, **`PasswordApi`**, into
  the `PasswordManager` core — same branch logic as Level 1, but every crossing is a port.
- Two extra driven ports appear: **`ResetTokenStore`** (save / findUsableByHash / consume)
  and **`Notifier`** (sendResetToken). The core never names a mail server or a DB table.
- Testability payoff: inject a **fake `Notifier`** that records the raw token, then a test
  can run `forgotPassword(…)` → read the captured token → `resetPassword(token, …)` and
  assert every credential was revoked — **no mail server, no DB**.
