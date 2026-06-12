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

interface PasswordApi:
    forgotPassword(identifier) -> GenericOk
    resetPassword(token, newPassword) -> void
    changePassword(accountId, current, newPassword, currentCredential) -> void
```

## Driven ports (what the core needs from the outside)

```
interface AccountRepository:  findByIdentifier(identifier) -> Account | null ;  save(account) -> Account
interface CredentialStore:    save(credential) ;  find(ref) -> Credential | null ;  revoke(ref) ;  revokeAllFor(accountId, exceptRef)
interface PasswordHasher:     hash(plain) -> Hash ;  verify(plain, hash) -> bool
interface RateLimiter:        check(identity, ip) -> Decision
interface AuditLog:           record(event, subject)
interface CredentialIssuer:   issue(account) -> Credential ;  revoke(credential)
interface ResetTokenStore:    save(token) ;  findUsableByHash(hash) -> ResetToken | null ;  consume(ref)
interface Notifier:           sendResetToken(account, rawToken)
```

## Core (implements AuthApi, depends only on ports)

```
class Authenticator implements AuthApi:
    constructor(accounts, credentials, hasher, rateLimiter, audit, issuers): ...
    // register / login / logout / identify bodies are exactly Level 1's pseudocode,
    // but every collaborator is a constructor-injected PORT, never a concrete class.

class PasswordManager implements PasswordApi:
    constructor(accounts, credentials, hasher, resetTokens, notifier, rateLimiter, audit): ...
    // forgot / reset / change bodies are exactly Level 1's pseudocode,
    // every collaborator a constructor-injected PORT (incl. ResetTokenStore, Notifier).
```

## Composition root (host application wires adapters → ports)

```
function buildAuth(db, cache, mailer):
    accounts    = DbAccountRepository(db)
    credentials = DbCredentialStore(db)
    hasher      = Argon2Hasher()
    rateLimiter = SharedCacheRateLimiter(cache)
    audit       = DbAuditLog(db)

    authn = Authenticator(
        accounts, credentials, hasher, rateLimiter, audit,
        issuers = { "web": SessionIssuer(...), "api": TokenIssuer(...) },
    )
    passwords = PasswordManager(
        accounts, credentials, hasher,
        resetTokens = DbResetTokenStore(db),
        notifier    = EmailNotifier(mailer),   // out-of-band adapter
        rateLimiter = rateLimiter, audit = audit,
    )
    return { authn, passwords }
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

## A second test — recovery flow with a fake notifier (no DB, no mail)

```
function test_reset_revokes_all_sessions():
    notifier = FakeNotifier()                 // captures (account, rawToken) in memory
    store    = InMemoryCredentialStore(seed = [ sessionFor("acc1"), tokenFor("acc1") ])
    pm = PasswordManager(
        accounts    = FakeAccountRepository(seed = [ Account(id="acc1", identifier="a@x.com") ]),
        credentials = store,
        hasher      = RealHasher(),
        resetTokens = InMemoryResetTokenStore(),
        notifier    = notifier,
        rateLimiter = AllowAllRateLimiter(),
        audit       = NullAuditLog(),
    )

    pm.forgotPassword("a@x.com")
    rawToken = notifier.lastTokenFor("acc1")  // the out-of-band secret, captured

    pm.resetPassword(rawToken, "new-strong-pass")

    assert store.activeCredentialsFor("acc1") == []          // every session/token revoked
    assert throws(BadRequest): pm.resetPassword(rawToken, "again")  // single-use
```

The out-of-band flow is fully exercised in memory — the **fake `Notifier` is the seam**
that makes "send an email" testable.
