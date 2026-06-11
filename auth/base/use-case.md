# Auth — Use Cases

Use cases derived from the three stories in `user-stories.md`. Level-independent.

```mermaid
---
title: Auth — Use Cases (web-only — register / login / logout)
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
    end

    Guest --> UC1
    Guest --> UC2
    User --> UC3
    Admin --> UC2
    Admin --> UC3

    UC2 -. "«include» verify credentials" .-> V(["Verify credentials"])
    UC2 -. "«include» issue credential" .-> I(["Issue session (web) / token (API)"])
```

| Use case | Actor(s) | Story | Outcome |
|---|---|---|---|
| Register | Guest | US-1 | account created (hashed password) |
| Login | Guest, Admin | US-2 | credential issued per channel |
| Logout | User, Admin | US-3 | session invalidated / token revoked |

Notes:
- **Login** «includes» *verify credentials* (one shared path) and *issue credential*,
  where issuance is the per-channel strategy fixed in `context.md`.
- Admin and User exercise the same use cases; they differ only by **authorization
  role** (separate component) — **not** by the auth flow or identity storage.
