# Auth — Domain Model

Class diagram for the Auth domain (level-independent). It fixes the *types and
relationships*; wiring and deployment are each level's concern.

```mermaid
---
title: Auth — Domain Model (class diagram)
---
classDiagram
    class Authenticator {
      +register(identifier, password) Account
      +login(identifier, password, channel) Credential
      +logout(credential) void
      +identify(request) AccountId
    }
    class Account {
      +id : OpaqueId
      +identifiers : Identifier[1..*]
      -passwordHash : Hash
      +status : Status
      +verify(password, hasher) bool
    }
    class Identifier {
      +type : Type
      +value : String
    }
    class PasswordHasher {
      <<interface>>
      +hash(plain) Hash
      +verify(plain, hash) bool
    }
    class CredentialIssuer {
      <<interface>>
      +issue(account, channel) Credential
      +revoke(credential) void
    }
    class SessionIssuer
    class TokenIssuer
    class Credential {
      <<abstract>>
      +id : OpaqueId
      +accountId : OpaqueId
      +expiresAt : Time
    }
    class Session
    class Token {
      +tokenHash : Hash
      +revokedAt : Time
    }
    class AccountRepository {
      <<interface>>
      +findByIdentifier(identifier) Account
      +save(account) void
    }
    class CredentialStore {
      <<interface>>
      +save(credential) void
      +find(ref) Credential
      +revoke(ref) void
    }
    class AuditLog {
      <<interface>>
      +record(event) void
    }
    class RateLimiter {
      <<interface>>
      +check(identity, ip) Decision
    }

    Authenticator --> AccountRepository
    Authenticator --> PasswordHasher
    Authenticator --> CredentialIssuer
    Authenticator --> CredentialStore
    Authenticator --> AuditLog
    Authenticator --> RateLimiter
    CredentialIssuer <|.. SessionIssuer
    CredentialIssuer <|.. TokenIssuer
    Credential <|-- Session
    Credential <|-- Token
    SessionIssuer ..> Session : creates
    TokenIssuer ..> Token : creates
    Account "1" o-- "1..*" Identifier : login handles
    Account ..> PasswordHasher : uses
```

Design notes:
- **Strategy + registry:** `CredentialIssuer` is chosen by channel from a registry
  (`web → SessionIssuer`, `api → TokenIssuer`). Adding a channel = register a new
  issuer, **not** edit `Authenticator` (OCP).
- **Interfaces at every boundary** (`*Repository`, `*Store`, `PasswordHasher`,
  `RateLimiter`) so Level 2 can swap implementations and Level 3 can move some behind
  the network — without touching `Authenticator`.
- `RateLimiter` and `AuditLog` are interfaces to *external / cross-cutting* providers.
- **Login identifier:** `findByIdentifier` resolves an account by email / username / NID.
  Whether identifiers are stored inline (A) or in a separate table (B) is the
  implementer's choice — see `data-model.md`.
