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

## Forgot + Reset (PasswordManager)

```mermaid
---
title: Auth — Level 1 Forgot + Reset (in-process — out-of-band send after commit)
---
sequenceDiagram
    autonumber
    actor C as Client
    participant Ctl as Controller
    participant P as PasswordManager
    participant RL as RateLimiter
    participant Repo as AccountRepository
    participant RT as ResetTokenStore
    participant N as Notifier
    participant H as PasswordHasher
    participant CS as CredentialStore
    participant Log as AuditLog

    rect rgb(245,245,245)
    Note over C,Log: Forgot — request a reset (generic reply, no enumeration)
    C->>Ctl: POST /forgot {identifier}
    Ctl->>P: forgotPassword(identifier)
    P->>RL: check(identifier, ip)
    P->>Repo: findByIdentifier(identifier)
    alt account exists
        P->>RT: save(hash(rawToken), expiresAt)
        P->>Log: record(password_reset_requested)
        Note right of P: commit first, then send
        P->>N: sendResetToken(account, rawToken)
    else no account
        Note right of P: do equal work, send nothing (timing-safe)
    end
    P-->>Ctl: generic ok
    Ctl-->>C: 200 "if that account exists, we've sent instructions"
    end

    rect rgb(245,245,245)
    Note over C,Log: Reset — consume the token (single-use)
    C->>Ctl: POST /reset {token, newPassword}
    Ctl->>P: resetPassword(token, newPassword)
    P->>RT: findUsableByHash(hash(token))
    alt token unusable (missing / expired / consumed)
        P-->>Ctl: Error(400 generic)
        Ctl-->>C: 400 invalid or expired token
    else usable + new password valid
        P->>H: hash(newPassword)
        P->>Repo: save(account with new hash)
        P->>RT: consume(token)
        P->>CS: revokeAllFor(accountId)
        P->>Log: record(password_reset)
        P-->>Ctl: ok
        Ctl-->>C: 200 reset done — please log in
    end
    end
```

## Change password (delta)
- `PasswordManager.changePassword(accountId, current, newPassword, currentCredential)`:
  load the account → `hasher.verify(current, account.hash)` (**re-authentication** —
  generic, timing-safe error on mismatch) → validate new (policy, **new ≠ old**) → in one
  transaction `hasher.hash`, `accounts.save`,
  `credentialStore.revokeAllFor(accountId, exceptRef = currentCredential)` →
  `audit.record(password_changed)`.
- Same in-process shape as login, but **no out-of-band channel** — the user is already
  authenticated; the proof is the **current password**, not a mailed token.
