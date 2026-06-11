# Auth — Level 1: Sequences

Login shown in full; register and logout follow the same in-process shape (deltas
noted below). **Every arrow is an in-process function call** — one app, one DB.

```mermaid
---
title: Auth — Level 1 Login (in-process calls)
---
sequenceDiagram
    autonumber
    actor C as Client
    participant Ctl as Controller (web/api)
    participant A as Authenticator
    participant RL as RateLimiter
    participant Repo as AccountRepository
    participant H as PasswordHasher
    participant I as CredentialIssuer
    participant CS as CredentialStore
    participant Log as AuditLog

    C->>Ctl: POST /login {identifier, password} (channel)
    Ctl->>A: login(identifier, password, channel)
    A->>RL: check(identifier, ip)
    alt rate limit exceeded
        A->>Log: record(login_blocked)
        A-->>Ctl: Error(429)
        Ctl-->>C: 429 Too Many Requests
    else allowed
        A->>Repo: findByIdentifier(identifier)
        A->>H: verify(password, hash)
        Note right of H: timing-safe — verify against<br/>a dummy hash if no account
        alt invalid credentials
            A->>Log: record(login_failure)
            A-->>Ctl: Error(401 generic)
            Ctl-->>C: 401 Invalid credentials
        else valid
            A->>I: issue(account, channel)
            alt channel = web
                I->>CS: save(session)
            else channel = api
                I->>CS: save(token hash)
            end
            A->>Log: record(login_success)
            A-->>Ctl: Credential
            alt channel = web
                Ctl-->>C: 200 + Set-Cookie (HttpOnly, Secure, SameSite)
            else channel = api
                Ctl-->>C: 200 {token}  (raw, shown once)
            end
        end
    end
    Note over A,CS: all in-process — transaction owned by Authenticator
```

## Register (delta)
- No rate-limit branch shown above is reused on register too (per-IP).
- `Authenticator.register`: validate input → check identifier uniqueness → `hasher.hash` →
  `accounts.save` in one transaction → `audit.record(register)`.
- Duplicate identifier → explicit `409 identifier already in use` (register may reveal this;
  login stays generic).

## Logout (delta)
- `Authenticator.logout(credential)`: `issuer.revoke(credential)` (invalidate session /
  revoke token row) in one transaction → `audit.record(logout)`.
- Subsequent requests carrying the old credential resolve to "unauthenticated" → 401.
