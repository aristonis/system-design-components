# Auth — User Stories & Requirements

Functional stories (register / login / logout) plus non-functional requirements,
shared by all three levels. Cross-cutting concerns (rate-limiting) are *referenced*,
not defined here — see `context.md`.

**Conventions:** story = *As a … I want … so that …*; acceptance criteria =
*Given / When / Then*. Actors **User** and **Admin** share these stories and
authenticate identically; admin privilege is an authorization concern (separate component).

---

## Functional stories

### US-1 — Register
*As a guest, I want to create an account with a login identifier (email / username / NID) + a password, so that I can access
the application.*

- **Given** a valid identifier and a policy-compliant password, **when** I register,
  **then** an account is created, the password is stored **hashed** (never plaintext),
  and its public id is **opaque and non-enumerable**.
- **Given** an identifier already registered, **when** I register, **then** I get an
  explicit *"identifier already in use"* error. *(Registration enumeration accepted as
  low-risk; login stays generic.)*
- **Given** an invalid identifier or a password failing the configured policy, **when** I
  register, **then** I get a field-level validation error and **no** account is created.

### US-2 — Login
*As a registered user, I want to log in with an identifier + password, so that I receive an
authenticated credential for my channel.*

- **Given** correct credentials via **browser**, **when** I log in, **then** a server
  session is created and an **HttpOnly + Secure + SameSite** cookie is set (CSRF on).
- **Given** correct credentials via **API**, **when** I log in, **then** a **bearer
  token** is issued (stored **hashed**, revocable) and returned **once**.
- **Given** incorrect credentials, **when** I log in, **then** I get a **generic 401**
  that does not reveal whether the account exists, and the check is **timing-safe**.
- **Given** repeated failures, **when** the Rate-Limiting component's limit is exceeded
  (per-identity **and** per-IP), **then** further attempts are rejected until the
  window resets. *(Thresholds owned by `rate-limiting/`.)*
- Login success and failure are written to the **audit log**.

### US-3 — Logout
*As an authenticated user, I want to log out, so that my credential can no longer be
used.*

- **Given** an active browser session, **when** I log out, **then** the session is
  invalidated server-side and the cookie cleared; reuse of the old session → **401**.
- **Given** an active API token, **when** I log out, **then** that token is revoked;
  reuse of it → **401**.
- Logout is written to the **audit log**.

---

## Non-functional requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-1 | Security | Passwords hashed with a slow KDF (argon2id/bcrypt); never logged or returned. |
| NFR-2 | Security | Login failures generic + timing-safe — **no user enumeration**. |
| NFR-3 | Security | Brute-force protection via the **Rate-Limiting** component (per-identity + per-IP). |
| NFR-4 | Security | Public ids **opaque, non-enumerable** (no raw incremental ids). |
| NFR-5 | Security | HTTPS only. Web: HttpOnly + Secure + SameSite cookies + CSRF. API: bearer token stored hashed, revocable. |
| NFR-6 | Security | All input validated at the boundary. |
| NFR-7 | Observability | Audit log for register, login (success/failure), logout. |
| NFR-8 | Performance | p95 auth request < 200 ms under normal load. |
| NFR-9 | Maintainability | One verify-credentials path; credential issuance is a per-channel strategy. |
| NFR-10 | Config | Password policy and token/session lifetime read from **config**, not hardcoded. |

---

## References (owned elsewhere)

- **Rate-Limiting** — thresholds & buckets live in the `rate-limiting/` component;
  listed as a dependency in `context.md`.
- **Password policy** — Auth-owned **config**; default *min 8 chars, no forced
  composition*; breach-list check deferred. Value in config, not in code.

## Deferred (out of base scope)

- Reset password, email verification, 2FA, social login.
- Authorization (roles/permissions) → separate `authorization/` component.
