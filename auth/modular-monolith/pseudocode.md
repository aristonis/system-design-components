# Auth — Level 1: Pseudocode

Language-agnostic. Level-1 wiring: every collaborator is **in-process**; the
`Authenticator` owns the transaction. Config values (password policy, lifetimes) are
read from config, not hardcoded.

## Credential issuer registry (strategy — OCP)

```
issuers = {
  "web": SessionIssuer,   // creates a server session, returns Session
  "api": TokenIssuer,     // creates a bearer token, returns Token (+ raw secret once)
}

function issuerFor(channel):
    issuer = issuers[channel]
    if issuer is null:
        fail "unsupported channel"        // fail loud, no silent default
    return issuer
```
Adding a channel (e.g. `mobile`) = insert one entry + one issuer class. The
`Authenticator` below never changes.

## Register

```
function register(identifier, password):
    validate(identifier)                              // boundary validation (per-type)
    if not passwordMeetsPolicy(password):             // policy from config
        fail ValidationError("password does not meet policy")
    transaction:                                      // atomic
        if accounts.findByIdentifier(identifier) exists:
            fail Conflict("identifier already in use") // explicit on register
        hash    = hasher.hash(password)
        account = accounts.save(Account(id = opaqueId(), identifier, hash, status = ACTIVE))
    audit.record(REGISTER, account.id)
    return account
```

## Login

```
function login(identifier, password, channel):
    if not rateLimiter.check(identity = identifier, ip = currentIp()).allowed:
        audit.record(LOGIN_BLOCKED, identifier)
        fail TooManyRequests()

    account = accounts.findByIdentifier(identifier)
    // timing-safe: always run a verify, even when the account is missing
    ok = hasher.verify(password, account?.passwordHash ?? DUMMY_HASH)

    if account is null or not ok or account.status != ACTIVE:
        audit.record(LOGIN_FAILURE, identifier)
        fail Unauthorized("invalid credentials")      // generic — no enumeration

    transaction:
        credential = issuerFor(channel).issue(account) // Session or Token
        credentialStore.save(credential)
    audit.record(LOGIN_SUCCESS, account.id)
    return credential                                  // controller sets cookie / returns token
```

## Logout

```
function logout(credential):
    transaction:
        issuerFor(credential.channel).revoke(credential) // invalidate session / revoke token
    audit.record(LOGOUT, credential.accountId)
```

## identify  (public facade — other modules ask "who is this request?")

```
function identify(request):
    ref      = readCredentialRef(request)              // cookie (web) or Bearer header (api)
    resolved = credentialStore.find(ref)
    if resolved is null or resolved.isExpired() or resolved.isRevoked():
        return null                                    // unauthenticated
    return resolved.accountId
```

## forgotPassword  (PasswordManager — generic + timing-safe, no enumeration)

```
function forgotPassword(identifier):
    if not rateLimiter.check(identity = identifier, ip = currentIp()).allowed:
        fail TooManyRequests()

    account  = accounts.findByIdentifier(identifier)
    rawToken = secureRandom()                          // high-entropy, shown once, sent out-of-band
    if account is not null:
        transaction:
            resetTokens.save(ResetToken(
                accountId  = account.id,
                tokenHash  = hashToken(rawToken),      // stored hashed (fast hash)
                expiresAt  = now() + RESET_TTL,        // from config
                consumedAt = null))
            audit.record(PASSWORD_RESET_REQUESTED, account.id)
        notifier.sendResetToken(account, rawToken)     // AFTER commit — never inside the txn
    // else: do equivalent work, send nothing (no enumeration)
    return GenericOk()                                 // identical reply either way
```

## resetPassword  (consume the token — single-use)

```
function resetPassword(rawToken, newPassword):
    record = resetTokens.findUsableByHash(hashToken(rawToken))  // not expired, not consumed
    if record is null:
        fail BadRequest("invalid or expired token")    // generic
    if not passwordMeetsPolicy(newPassword):
        fail ValidationError("password does not meet policy")

    transaction:
        account = accounts.findById(record.accountId)
        account.passwordHash = hasher.hash(newPassword)
        accounts.save(account)
        resetTokens.consume(record.id)                  // single-use — cannot be replayed
        credentialStore.revokeAllFor(account.id, exceptRef = null) // eject every session/token
        audit.record(PASSWORD_RESET, account.id)
    // no auto-login — client logs in fresh (config-overridable)
```

## changePassword  (authenticated — re-verify current password)

```
function changePassword(accountId, currentPassword, newPassword, currentCredential):
    account = accounts.findById(accountId)
    if not hasher.verify(currentPassword, account.passwordHash):   // re-authentication
        audit.record(PASSWORD_CHANGE_FAILED, accountId)
        fail Unauthorized("current password is incorrect")          // generic, timing-safe
    if newPassword == currentPassword or not passwordMeetsPolicy(newPassword):
        fail ValidationError("choose a different, policy-compliant password")

    transaction:
        account.passwordHash = hasher.hash(newPassword)
        accounts.save(account)
        credentialStore.revokeAllFor(accountId, exceptRef = currentCredential) // keep this session
        audit.record(PASSWORD_CHANGED, accountId)
```

Constants/config: `DUMMY_HASH` (fixed valid-shaped hash for timing safety),
`passwordMeetsPolicy` (min length etc. from config), credential lifetimes (from config),
`RESET_TTL` (reset-token lifetime, from config). `hashToken()` is a **fast** hash
(e.g. SHA-256), not the slow password KDF: the reset token is already high-entropy, so it
needs no key-stretching — and storing only its hash means a leaked `PASSWORD_RESET` row
can't be replayed.
