# Auth — User Stories & Requirements

Functional stories (register / login / logout, plus password recovery & change) and
non-functional requirements, shared by all three levels. Cross-cutting concerns
(rate-limiting, out-of-band notification) are *referenced*, not defined here — see
`context.md`.

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

### US-4 — Forgot password (request a reset)
*As a user who can't log in, I want to request a password reset with my identifier, so
that I can regain access without knowing my old password.*

- **Given** any identifier, **when** I request a reset, **then** I get the **same generic
  response** (*"if that account exists, reset instructions have been sent"*) — the reply
  **never reveals whether the account exists** (no enumeration).
- **Given** an existing account, **when** I request a reset, **then** a **single-use,
  time-limited** reset token is generated, stored **hashed**, and sent **out-of-band**
  (email / SMS) — it is **never** returned in the HTTP response.
- **Given** an unknown identifier, **when** I request a reset, **then** equivalent work is
  done (**timing-safe**) and nothing is sent.
- **Given** repeated requests, **when** the Rate-Limiting limit (per-identity **and**
  per-IP) is exceeded, **then** further requests are rejected. *(Hard limit — this
  endpoint sends messages and costs money.)*
- Logged to the **audit log** (`password_reset_requested`).

### US-5 — Reset password (consume the token)
*As a user holding a valid reset token, I want to set a new password, so that I can log
in again.*

- **Given** a token that **exists, matches (timing-safe), is unexpired and unconsumed**,
  and a policy-compliant new password, **when** I reset, **then** the password hash is
  replaced, the token is **consumed (single-use)**, and **all** existing sessions/tokens
  are **revoked**.
- **Given** an unknown, expired, or already-used token, **when** I reset, **then** I get a
  **generic** *"invalid or expired token"* error and nothing changes.
- **Given** a new password failing policy, **when** I reset, **then** I get a validation
  error and nothing changes.
- **Given** a successful reset, **then** I am **not** auto-logged-in — I must log in fresh
  *(config-overridable)*.
- Logged to the **audit log** (`password_reset`).

### US-6 — Change password (authenticated)
*As an authenticated user, I want to change my password by confirming my current one, so
that a stolen session alone cannot change it.*

- **Given** my **correct current password** and a policy-compliant new one **different
  from the current**, **when** I change it, **then** the hash is replaced and **all
  other** sessions/tokens are revoked while **my current credential stays valid**.
- **Given** an **incorrect current password**, **when** I change it, **then** I get a
  generic error (**timing-safe**) and nothing changes.
- **Given** a new password equal to the current one or failing policy, **when** I change
  it, **then** I get a validation error and nothing changes.
- Logged to the **audit log** (`password_changed`).

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
| NFR-7 | Observability | Audit log for register, login (success/failure), logout, and password reset-request / reset / change. |
| NFR-8 | Performance | p95 auth request < 200 ms under normal load. |
| NFR-9 | Maintainability | One verify-credentials path; credential issuance is a per-channel strategy. |
| NFR-10 | Config | Password policy and token/session lifetime read from **config**, not hardcoded. |
| NFR-11 | Security | Reset tokens single-use, time-limited (config TTL), stored **hashed** at rest, delivered **out-of-band** only — never in a response body. |
| NFR-12 | Security | Forgot-password is generic + timing-safe — **no account enumeration**; out-of-band send only when the account exists. |
| NFR-13 | Security | Reset revokes **all** sessions/tokens; change-password revokes **all others** — containment after a credential change. |
| NFR-14 | Security | Change-password **re-verifies the current password** (re-authentication), independent of the active session. |

---

## References (owned elsewhere)

- **Rate-Limiting** — thresholds & buckets live in the `rate-limiting/` component;
  listed as a dependency in `context.md`.
- **Password policy** — Auth-owned **config**; default *min 8 chars, no forced
  composition*; breach-list check deferred. Value in config, not in code. Reused by
  register, reset, and change.
- **Notifier (out-of-band)** — delivery of the reset link / OTP via email / SMS is owned
  by a notification component; Auth mints the token and asks it to send. Referenced, not
  built here.

## Deferred (out of base scope)

- Email verification, 2FA, social login.
- Authorization (roles/permissions) → separate `authorization/` component.
