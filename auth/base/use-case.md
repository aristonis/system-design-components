# Auth — Use Cases

Use cases derived from the three stories in `user-stories.md`. Level-independent.

```mermaid
---
title: Auth — Use Cases (register / login / logout / password recovery / change)
---
flowchart LR
    Guest(["Guest"])
    User(["User"])
    Admin(["Admin"])

    subgraph System["Auth component"]
      direction TB
      UC1(["Register"])
      UC2(["Login"])
      UC3(["Logout"])
      UC4(["Forgot password"])
      UC5(["Reset password"])
      UC6(["Change password"])
    end

    Guest --> UC1
    Guest --> UC2
    Guest --> UC4
    Guest --> UC5
    User --> UC3
    User --> UC6
    Admin --> UC2
    Admin --> UC3
    Admin --> UC6

    UC2 -. "«include» verify credentials" .-> V(["Verify credentials"])
    UC2 -. "«include» issue credential" .-> I(["Issue session (web) / token (API)"])
    UC5 -. "«include» revoke all credentials" .-> R(["Revoke sessions / tokens"])
    UC6 -. "«include» verify credentials" .-> V
    UC6 -. "«include» revoke other credentials" .-> R
```

| Use case | Actor(s) | Story | Outcome |
|---|---|---|---|
| Register | Guest | US-1 | account created (hashed password) |
| Login | Guest, Admin | US-2 | credential issued per channel |
| Logout | User, Admin | US-3 | session invalidated / token revoked |
| Forgot password | Guest | US-4 | reset token sent out-of-band; generic reply (no enumeration) |
| Reset password | Guest | US-5 | password replaced, token consumed, **all** credentials revoked |
| Change password | User, Admin | US-6 | password replaced, **other** credentials revoked |

Notes:
- **Login** «includes» *verify credentials* (one shared path) and *issue credential*,
  where issuance is the per-channel strategy fixed in `context.md`.
- **Forgot / Reset** is the **unauthenticated recovery path** — the actor is a Guest
  (locked out). The out-of-band token *is* the proof of identity; on reset, credentials
  are revoked so any old session/token is ejected.
- **Change password** is the **authenticated path** — it «includes» *verify credentials*
  (re-checks the **current** password, not just the session) and revokes the other
  credentials, keeping the current one.
- Admin and User exercise the same use cases; they differ only by **authorization
  role** (separate component) — **not** by the auth flow or identity storage.
