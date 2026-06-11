# Auth — Level 3: Sequences

Two flows that capture what changed: **login is now a network call**, and **validating
a request needs no call to auth at all**.

## Login + refresh (network calls to auth-service)

```mermaid
---
title: Auth — Level 3 Login & Refresh (network)
---
sequenceDiagram
    autonumber
    actor C as Client / BFF
    participant GW as API Gateway
    participant A as auth-service
    participant DB as auth-db
    participant K as Signing key (private)

    C->>GW: POST /login {identifier, password}
    GW->>A: login(...)
    A->>DB: findByIdentifier + verify (core, timing-safe)
    alt invalid
        A-->>GW: 401 generic
        GW-->>C: 401
    else valid
        A->>K: sign access JWT {sub, exp, jti}
        A->>DB: save refresh token (hashed)
        A-->>GW: { access JWT, refresh token }
        GW-->>C: 200 tokens
    end

    Note over C,A: later, when the access token expires
    C->>GW: POST /refresh {refresh token}
    GW->>A: refresh(...)
    A->>DB: validate refresh (exists, not expired, not revoked)
    A->>K: sign new access JWT
    A-->>GW: { access JWT }
    GW-->>C: 200
```

## Authenticated request to another service (NO call to auth)

```mermaid
---
title: Auth — Level 3 Request Validation (local, stateless)
---
sequenceDiagram
    autonumber
    actor C as Client / BFF
    participant GW as API Gateway
    participant O as order-service
    participant J as JWKS cache (public key)

    C->>GW: GET /orders  (Authorization: Bearer access JWT)
    GW->>O: forward request
    O->>J: public key for kid (cached, refreshed periodically)
    O->>O: verify signature + exp + claims  (LOCAL, no network to auth)
    alt valid
        O-->>C: 200  (identity = jwt.sub)
    else invalid/expired
        O-->>C: 401
    end
    Note over O,J: contrast L1/L2, where "who is this" was a DB/port lookup.<br/>Here it is a local signature check — no auth-service call.
```

The second diagram is the whole point of Level 3: identity verification **scales with
each service** and does not funnel back to a single auth datastore.
