# Auth — Domain Model

**Module:** `apps/api/src/modules/auth` (+ Better Auth integration)
**Status:** Draft v0.1 — 2026-08-12
**PRD source:** §5.1 Authentication · §5.11 Settings (account/security portions)

---

## 1. Overview & Scope

Auth owns **identity**: who the user is, how they prove it, and how that proof stays valid across requests.

**In scope:**
- Email/password registration, email verification, Google/GitHub OAuth
- Login, logout, session management
- Password reset, password change
- Account email change (with re-authentication + verification)
- The authenticated identity context consumed by every other module

**Out of scope (owned by other modules):**
- Workspace membership, roles, invitations → `members` module (auth only guarantees *verified identity*; invitations/roles decide *authorization within a workspace*)
- Workspace onboarding → `workspace` module
- Display name/avatar/theme editing surfaces → `users`/`settings` (the User entity carries the base attributes; editing flows live in settings docs)

**Key separation:** Auth = **authentication** (who you are). Authorization (what you can do) = `members` + shared permissions layer. The two are never mixed.

---

## 2. Domain Entities

### 2.1 User (the account)

The single identity a person owns across all workspaces.

| Attribute | Notes |
|---|---|
| `email` | Unique across all users. The identity key. |
| `emailVerified` | Whether the email is proven controlled. **Gates protected access for email/password users.** |
| `passwordHash` | Only for email/password users; never plaintext; hashed by Better Auth (argon2). Null for OAuth-only users. |
| `name` | Display name (profile attribute, edited in settings). |
| `image` | Avatar URL (R2) — profile attribute. |
| `theme` | Account-level preference (light/dark/system) — settings domain. |

**Invariants:**
- One user per unique email.
- A user may hold both email/password credentials **and** one or more OAuth identities, as long as they resolve to the same verified email.
- Email/password users exist in an **unverified** state until their email is verified; OAuth-created users are always verified (provider-verified email).

### 2.2 Account (OAuth identity)

A link between a User and an external provider identity.

| Attribute | Notes |
|---|---|
| `provider` | `google` or `github` (MVP). |
| `providerAccountId` | The user's id at the provider. |
| `provider email` | Must be provider-verified; satisfies Shipyard's verification requirement. |

**Invariants:**
- A provider identity links to **at most one** Shipyard user.
- A verified provider email matching an existing user resolves to **that user** — never a duplicate account.
- OAuth access tokens/secrets are never exposed client-side.

### 2.3 Session

The authentication proof the browser holds (HttpOnly cookie).

**Invariants:**
- Sessions persist across browser refreshes.
- Expired sessions are rejected → user redirected to login.
- Logout invalidates the current session **immediately**, without confirmation.
- A user may hold multiple sessions (multiple devices) — one per login.

### 2.4 VerificationToken

Single-use, expiring capability tokens used for: email verification, password reset, and account email change.

**Invariants:**
- Single-use: consumed on success; invalid/expired/already-used tokens show a recoverable result with a path forward (request new link / continue to login).
- Expire after a configurable period.
- Resending is rate-limited and never creates duplicate accounts.

---

## 3. Identity & Verification Model

Shipyard has **one rule for identity**: *an account requires a verified email address.*

```
                    ┌──────────────────────────────┐
                    │        VERIFIED EMAIL        │
                    │   (the single identity key)  │
                    └──────────────┬───────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
   email/password            Google OAuth           GitHub OAuth
   (unverified until         (provider-verified     (provider-verified
    link clicked)             email accepted)        email accepted)
              │                    │                    │
              └────────────────────┴────────────────────┘
                                   │
                     resolves to ONE User account
                     (merge by verified email — no duplicates)
```

**Verification state machine (email/password users):**

```
registered ──▶ UNVERIFIED ──(verification link valid)──▶ VERIFIED ──▶ full access
                  │
                  └──(login attempt)──▶ verification-pending screen (can resend)
```

- Unverified users can resend (rate-limited) but cannot access protected product areas.
- A pending account email change does **not** revoke the current verified email: the old email stays active until the new one is verified.

---

## 4. Domain Invariants (Business Rules)

From PRD §5.1 (business rules + edge cases), condensed to domain-level rules:

1. Emails are unique; a user may belong to multiple workspaces (membership ≠ identity).
2. Passwords are never stored in plain text.
3. Email/password users require a verified email before any protected access.
4. Provider-verified emails satisfy the verification requirement; provider identity resolves by email (no duplicate accounts).
5. Verification and password-reset tokens are single-use, expiring, and resendable only with rate limiting.
6. Logout invalidates the session immediately; no confirmation step.
7. Expired/invalid sessions redirect to login; protected pages require a valid session.
8. Password reset: link expires; the reset request response is generic (does not reveal whether an email has an account).
9. Changing account email requires recent re-authentication (current password, or provider re-auth for OAuth-only users) **and** verification of the new address before it replaces the current one.
10. Changing password requires the current password.
11. OAuth failures (cancel, provider error, missing verified email) return the user to the auth screen with a clear retryable message; **no account/session is created on failure**.
12. Authentication is independent of workspace permissions (authn ≠ authz).

---

## 5. Domain Operations

The operations auth exposes to the rest of the system (endpoints + request lifecycle in `03-request-lifecycle.md` and `04-api-design.md`):

| Operation | Description | Requires |
|---|---|---|
| `register` | Create user (email/password) → unverified + verification email | — |
| `verifyEmail` | Consume token → mark verified → sign in → onboarding/invitation | valid token |
| `login` | Email/password or OAuth → session | verified account |
| `oauthAuthorize` | Google/GitHub authorization → resolve/create user → session | provider verified email |
| `logout` | Invalidate session | session |
| `validateSession` | Confirm session for every protected request | session |
| `requestPasswordReset` | Send reset email (generic response) | email |
| `resetPassword` | Consume token → set new password | valid token |
| `changePassword` | Set new password | current password + session |
| `requestEmailChange` | Re-auth + send verification to new address | session + re-auth |
| `confirmEmailChange` | Consume token → swap email | valid token |
| `resendVerification` | Re-send verification email (rate-limited) | unverified user |

---

## 6. Relationships to Other Modules

```
                        ┌──────────┐
                        │  User    │  ← owned by auth (identity)
                        └────┬─────┘
        ┌────────────────────┼─────────────────────────┐
        ▼                    ▼                         ▼
  members module      workspace module           notifications module
  (memberships,       (creator → owner,         (recipient of
   roles, invites)     onboarding entry)         assignment/mention)
        │                    │
        └──── auth provides ─┘
             VERIFIED IDENTITY + SESSION CONTEXT
             (every module consumes: req.user + req.session)
```

- **Consumed by all modules:** the authenticated context (`userId`, `sessionId`) resolved once per request by the auth middleware — the trust anchor for workspace scoping and RBAC.
- **Invitations:** pending invitations are shown only to *verified* users (auth guarantees); accept/decline logic lives in `members`.
- **Onboarding:** post-verification/OAuth redirect goes to pending invitation or workspace onboarding (`workspace`).

---

## 7. Trust Boundaries & Security Properties

1. All auth input is untrusted until validated (Zod at the API edge).
2. The session cookie is HttpOnly and forwarded server-side only (Next.js proxy — never exposed to browser JS).
3. Verification tokens are capability tokens: possession proves email control; single-use + expiry bounds the window.
4. OAuth state/flow is handled by Better Auth; provider tokens stored server-side, never in client-visible content.
5. Auth endpoints are rate-limited (login, register, resend, password reset) to blunt brute force and abuse.
6. Failed auth responses are generic where revealing account existence would help attackers (login, password reset).
7. Session validation runs on **every** protected request — no client-trusted "logged in" flags.

---

## 8. Non-Goals (MVP)

Per PRD future enhancements — intentionally excluded: passkeys, 2FA, SSO, device/session management UI, email/push notifications for auth events, account deletion flows, OAuth for additional providers.

---

## 9. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Session lifetime policy (fixed expiry vs rolling/sliding)? | Better Auth defaults; decide in data model doc |
| 2 | Password policy specifics (min length, complexity)? | Better Auth defaults (8+); confirm if stricter |
| 3 | Rate-limit thresholds per auth endpoint? | Decide in API design doc |
| 4 | Multiple sessions per user allowed (default yes) — any MVP cap? | Better Auth default: unlimited |
