# Auth — Glossary

Shared terms (ubiquitous language) for the Auth component. One definition each.

| Term | Meaning |
|---|---|
| **Account** | A registered identity that can authenticate (login identifier + password hash). User and Admin are account kinds. |
| **Login identifier** | A handle used to find an account at login — email, username, NID, … (its `type`). Generalizes "email". |
| **Authentication** | Proving *who you are*. Auth's job. |
| **Authorization** | Deciding *what you may do* (roles/permissions). Separate component. |
| **Channel** | How a client reaches the app: **web** (browser) or **api** (SPA/mobile). Selects the credential type. |
| **Credential** | Proof of an authenticated identity issued at login. Concrete forms: Session, Token. |
| **Session** | Server-side auth state for the web channel, carried in an HttpOnly cookie. |
| **Token (bearer)** | Opaque string for the API channel, sent as `Authorization: Bearer …`; stored hashed, revocable. |
| **Credential Issuer** | Strategy that creates a Credential for one channel (SessionIssuer / TokenIssuer). |
| **Password hashing (KDF)** | One-way slow hash (argon2id/bcrypt) used to store and verify passwords. |
| **CSRF** | Attack abusing cookie auth; mitigated for the web channel, not needed for bearer tokens. |
| **User enumeration** | Leaking whether an account exists via different responses/timing. Prevented on login. |
| **Opaque id** | A public identifier that is non-guessable and non-sequential. |
| **Rate limiting** | Throttling attempts per identity / IP. Owned by the `rate-limiting/` component. |
| **Audit log** | Record of auth events (register / login / logout / password reset-request / reset / change) for observability. |
| **Identity provider** | The source of identities the system authenticates against (the account store). |
| **PasswordManager** | The service owning forgot / reset / change password; sibling to the Authenticator, sharing the same boundary ports plus `ResetTokenStore` + `Notifier`. |
| **Account recovery** | Regaining access *without* the old password, via an out-of-band proof: forgot (request) → reset (consume). |
| **Reset token** | A single-use, time-limited recovery secret bound to one account; stored **hashed**, delivered out-of-band, consumed on first use. |
| **Out-of-band channel** | A delivery path separate from the login channel (email / SMS) used only to hand the user a recovery secret; never carries a password. |
| **Re-authentication** | Re-proving identity with the **current password** for a sensitive action (change-password), even inside an active session. |
| **Single-use** | Valid for exactly one successful use, then consumed — the reset token's lifecycle. |
