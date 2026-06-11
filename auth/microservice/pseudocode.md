# Auth — Level 3: Pseudocode

The credential-verification **core is still the same** (from Level 1). What is new:
**signing JWTs**, **refresh tokens**, and **local validation in every other service**.
All values (TTLs, key ids) come from config.

## auth-service — register

```
function register(identifier, password):
    validate(identifier)                              // per-type validator
    if not passwordMeetsPolicy(password): fail ValidationError
    transaction:                                      // atomic, in auth-db
        if accounts.findByIdentifier(identifier) exists:
            fail Conflict("identifier already in use")
        account = accounts.save(Account(id = opaqueId(), identifier, hasher.hash(password), ACTIVE))
    audit(REGISTER, account.id)
    publish(UserRegistered{ accountId: account.id })  // optional: notify other services
    return account
```

## auth-service — login

```
function login(identifier, password):
    if not rateLimiter.check(identifier, currentIp()).allowed:
        audit(LOGIN_BLOCKED, identifier); fail TooManyRequests()

    account = accounts.findByIdentifier(identifier)
    ok = hasher.verify(password, account?.passwordHash ?? DUMMY_HASH)   // timing-safe
    if account is null or not ok or account.status != ACTIVE:
        audit(LOGIN_FAILURE, identifier); fail Unauthorized("invalid credentials")

    access  = signJwt(
        header = { alg: "EdDSA", kid: CURRENT_KID },
        claims = { sub: account.id, iat: now(), exp: now() + ACCESS_TTL, jti: opaqueId() },
        key    = PRIVATE_KEY)
    refresh = opaqueToken()
    refreshTokens.save(hash(refresh), account.id, exp = now() + REFRESH_TTL)

    audit(LOGIN_SUCCESS, account.id)
    return { access, refresh, expiresIn: ACCESS_TTL }
```

## auth-service — refresh (access token expired)

```
function refresh(refreshToken):
    row = refreshTokens.find(hash(refreshToken))
    if row is null or row.expired() or row.revoked:
        fail Unauthorized("invalid refresh token")
    // optional rotation: revoke old, issue new refresh
    access = signJwt({ sub: row.accountId, iat: now(), exp: now() + ACCESS_TTL, jti: opaqueId() },
                     PRIVATE_KEY)
    return { access, expiresIn: ACCESS_TTL }
```

## auth-service — logout / revoke

```
function logout(refreshToken, accessJti, accessExp):
    refreshTokens.revoke(hash(refreshToken))
    denylist.add(accessJti, ttl = accessExp - now())   // optional: instant access revoke
    audit(LOGOUT, ...)
```

## any service — validation middleware (LOCAL, no call to auth)

```
function authenticate(request):
    token = bearer(request)                       // from header (or BFF cookie)
    key   = jwksCache.publicKeyFor(token.header.kid)   // cached; refreshed periodically
    jwt   = verifySignature(token, key)           // no network to auth-service
    if jwt is invalid or jwt.exp < now():
        fail Unauthorized()
    if denylistEnabled and denylist.contains(jwt.jti):
        fail Unauthorized()
    return jwt.sub                                // accountId — identity only
```

Notes:
- `verifySignature` + `jwksCache` make identity checks **local and cheap** in every
  service — the core scalability win of Level 3.
- The token carries **identity** (`sub`), not authorization. Roles/permissions are the
  separate Authorization component's responsibility.
- Config: `ACCESS_TTL` (short, e.g. 5–15 min), `REFRESH_TTL` (longer), `CURRENT_KID`,
  JWKS refresh interval, whether the denylist is enabled.
- **Through-line:** `authenticate(request) → sub` is L3's form of L1/L2
  `identify(request) → AccountId`; the stored credential that was `API_TOKEN` in `base/`
  is the `REFRESH_TOKEN` here, while the access credential is now a stateless JWT.
