# Auth — Level 2: Pseudocode

The **core logic is unchanged** from Level 1 (`modular-monolith/pseudocode.md`). What
Level 2 adds is **explicit ports** and a **composition root** that injects adapters.

## Driving port (the package's public contract)

```
interface AuthApi:
    register(identifier, password) -> Account
    login(identifier, password, channel) -> Credential
    logout(credential) -> void
    identify(request) -> AccountId | null
```

## Driven ports (what the core needs from the outside)

```
interface AccountRepository:  findByIdentifier(identifier) -> Account | null ;  save(account) -> Account
interface CredentialStore:    save(credential) ;  find(ref) -> Credential | null ;  revoke(ref)
interface PasswordHasher:     hash(plain) -> Hash ;  verify(plain, hash) -> bool
interface RateLimiter:        check(identity, ip) -> Decision
interface AuditLog:           record(event, subject)
interface CredentialIssuer:   issue(account) -> Credential ;  revoke(credential)
```

## Core (implements AuthApi, depends only on ports)

```
class Authenticator implements AuthApi:
    constructor(accounts, credentials, hasher, rateLimiter, audit, issuers): ...
    // register / login / logout / identify bodies are exactly Level 1's pseudocode,
    // but every collaborator is a constructor-injected PORT, never a concrete class.
```

## Composition root (host application wires adapters → ports)

```
function buildAuth(db, cache):
    return Authenticator(
        accounts    = DbAccountRepository(db),
        credentials = DbCredentialStore(db),
        hasher      = Argon2Hasher(),
        rateLimiter = SharedCacheRateLimiter(cache),
        audit       = DbAuditLog(db),
        issuers     = { "web": SessionIssuer(...), "api": TokenIssuer(...) },
    )
```

## Why this matters — a test with fakes (no DB)

```
function test_login_rejects_bad_password():
    auth = Authenticator(
        accounts    = FakeAccountRepository(seed = [ Account(identifier="a@x.com", hash=H("right")) ]),
        credentials = InMemoryCredentialStore(),
        hasher      = RealHasher(),
        rateLimiter = AllowAllRateLimiter(),
        audit       = NullAuditLog(),
        issuers     = { "api": TokenIssuer(...) },
    )
    assert throws(Unauthorized):
        auth.login("a@x.com", "wrong", "api")
```

The core is exercised with **zero infrastructure** — the proof that the boundary is
real. Swapping `DbAccountRepository` for a different store later means writing one new
adapter; the core does not change (OCP / DIP).
