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
