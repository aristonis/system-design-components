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
| **Audit log** | Record of auth events (register / login / logout) for observability. |
| **Identity provider** | The source of identities the system authenticates against (the account store). |
