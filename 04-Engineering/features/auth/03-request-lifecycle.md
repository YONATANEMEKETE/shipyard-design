# Auth — Request Lifecycle

**Module:** `apps/api/src/modules/auth`
**Status:** Draft v0.1 — 2026-08-12
**Relies on:** ADR-003 (Next.js proxy, internal API) · `02-data-model.md` · `04-api-design.md`

---

## 1. Overview — two request classes

Auth traffic comes in two shapes with different lifecycles:

| Class | Examples | Transport |
|---|---|---|
| **JSON API flows** | register, login, logout, get-session, change-password, resend | Browser → fetch (via Next proxy) → JSON response |
| **Redirect flows** | OAuth, verify-email link, password-reset link, email-change link | Browser → HTTP redirects (provider ↔ app ↔ result screens) |

Both share the same global pipeline and the same session cookie; only the response transport differs.

---

## 2. The global pipeline (every auth request)

```
Browser
  │  https://shipyard.yonatanem.com/api/v1/auth/...
  ▼
Caddy (:443, TLS) ──▶ Next.js (:3000)
  │  rewrite /api/* → http://api:4000/api/*      [internal network]
  │  (Set-Cookie from the API is passed through unchanged — same origin)
  ▼
Express API (:4000)
  ├── request-id middleware        → assigns X-Request-Id, logs start
  ├── Pino structured log          → method, path, request-id, ip, user-agent
  ├── security headers             → Helmet-equivalent (no CORS: no cross-origin calls exist)
  ├── rate-limit middleware        → per-IP / per-email limits (api-design §5)
  ├── body parsing (JSON, size-capped)
  └── Better Auth handler          → /api/v1/auth/* routes
        │
        ├── cookie parse → session lookup → (hashed) token check
        ├── Zod validation (request body/query)
        ├── domain logic (register/login/reset/...)
        ├── Prisma writes (transactional where multi-step)
        └── response + Set-Cookie (new/updated session cookie)
  ▼
Response back through the proxy → browser stores HttpOnly cookie
```

**Trust boundary recap:** the browser only ever sees Next.js. The session cookie is created by the API, forwarded through the proxy, and stays HttpOnly — JS never touches it.

---

## 3. Flow — Register (JSON)

```
1. POST /api/v1/auth/sign-up/email
   body: { name, email, password }            [Zod: name 1-64, email normalized, pwd ≥ 8]
2. rate limit check (5/min/IP)
3. email uniqueness check                    → 400 AUTH_EMAIL_IN_USE
4. TRANSACTION:
   a. INSERT User (emailVerified = false)
   b. INSERT Account (providerId = "credential", password = argon2 hash)
5. INSERT Verification (identifier: "email-verification:{email}", value: hashed token,
   expiresAt: now + 24h)                     [not in the user transaction — token can
                                              fail independently of account creation]
6. Send email via Resend (verification link → /verify-email?token=...&callbackURL=...)
7. 201 → web shows "Verification Pending" screen (destination email visible)
   [no session cookie set — protected access requires verification]
```

**On email send failure:** the account exists but the user is unverified → the pending screen offers "resend" (rate-limited). No duplicate accounts are ever created.

---

## 4. Flow — Email verification (redirect)

```
1. User clicks the link: /verify-email?token=T&callbackURL=...
   (token arrives at the WEB app — same-origin, so the browser sends it with the request)
2. Web route handler forwards to API: GET /api/v1/auth/verify-email?token=T
3. API:
   a. Lookup Verification by identifier + value
   b. token used?  → 404/AUTH_TOKEN_INVALID
   c. expired?     → 404/AUTH_TOKEN_INVALID
   d. valid → DELETE Verification row (single-use consumed)
   e. TRANSACTION: UPDATE User SET emailVerified = true
   f. create session → Set-Cookie
4. Redirect (302) → callbackURL (pending invitation or workspace onboarding)
```

**Failure paths** (invalid/expired/used): redirect to `Auth / Email Verification / Invalid Token` screen with two actions — "request a new link" (resend, rate-limited) and "go to login". Always recoverable; never a dead end.

---

## 5. Flow — Login & session establishment (JSON)

```
1. POST /api/v1/auth/sign-in/email
   body: { email, password, rememberMe? }     [Zod]
2. rate limit check (10/min/IP)
3. credential lookup (Account where providerId="credential" + email → user)
4. argon2 verify → fail: 400 AUTH_INVALID_CREDENTIALS (generic)
5. emailVerified == false → 403 AUTH_EMAIL_NOT_VERIFIED
   → web shows Verification Pending (resend available)
6. INSERT Session (token hashed at rest, expiresAt = +7d, or +30d if rememberMe)
7. Response 200 { user, session } + Set-Cookie: better-auth.session_token
   (HttpOnly, SameSite=Lax, Secure in prod, Path=/)
8. Web redirects by workspace state (PRD): pending invitation → onboarding
   → sole workspace dashboard → workspace selection
```

**Cookie mechanics through the proxy (the detail that makes ADR-003 work):**
- The API's `Set-Cookie` header is passed through Next's rewrite untouched — the browser sees it as **same-origin** (`shipyard.yonatanem.com`), so the cookie is accepted and stored.
- Subsequent requests send the cookie with every `/api/*` call; Next forwards the header; the API reads it. No domain juggling, no CORS.

---

## 6. Flow — OAuth (redirect chain)

```
1. GET /api/v1/auth/sign-in/social?provider=google|github
2. 302 → provider authorization page (state param protects CSRF)
3. User authorizes → provider redirects to callback:
   GET /api/v1/auth/callback/google?code=...&state=...
4. API exchanges code for tokens; provider MUST return a verified email
   a. verified email exists  → resolve user (merge — no duplicate)
   b. verified email new     → INSERT User (emailVerified = true) + Account (provider)
   c. no verified email      → 302 → auth screen + AUTH_OAUTH_FAILED (retryable)
5. INSERT Session → Set-Cookie (same mechanics as login)
6. 302 → pending invitation / onboarding / workspace (same routing as login)
```

**Failure** (cancel, provider down, no verified email): redirect back to the auth screen with a clear retryable message. **No account, no session** is created on any failure path (domain rule 11).

---

## 7. Flow — Every protected request (the hot path)

This runs on **every** request to any protected route in every module:

```
Browser ──▶ Next (server component / route handler / client fetch with cookie)
  └──▶ API: requireSession guard
        ├── cookie present?          no  → 401 AUTH_UNAUTHORIZED → web → /login
        ├── session lookup (hashed token) → not found → 401 (lazy-delete stale row)
        ├── expiresAt < now?         yes → 401 + row deleted (lazy cleanup)
        ├── attach req.user + req.session
        └── next: requireWorkspaceMember → requireRole(...)   [per route, PRD matrix]
```

**Expiry handling:** the web middleware calls `get-session` through the proxy before rendering protected pages; an expired/invalid session redirects to `/login` (PRD). Server-side, the same check is the `requireSession` guard — no client-trusted "logged in" state anywhere.

**Cookie rotation:** Better Auth refreshes the session token periodically (rolling token, fixed expiry per data model §8) — the response's `Set-Cookie` is applied the same way as login.

---

## 8. Flow — Logout (JSON)

```
1. POST /api/v1/auth/sign-out          (session cookie required)
2. DELETE Session row (current token)
3. 204 + Set-Cookie: clear cookie (expires in the past)
4. Web redirects to /login — no confirmation step (PRD)
```

Already-expired session: treated as success (204) — logout is idempotent from the user's perspective.

---

## 9. Flow — Password reset (two steps)

```
REQUEST:
1. POST /api/v1/auth/request-password-reset { email }   [rate limit 3/h/email]
2. ALWAYS: 200 (generic — no existence leak; PRD rule)
   + if account exists: INSERT Verification (reset token, +1h) → Resend email
   → web shows "Email Sent" screen (design: Auth / Forgot Password / Email Sent)

CONFIRM:
3. User clicks link → web /reset-password?token=T
4. POST /api/v1/auth/reset-password { newPassword, token }   [Zod: pwd ≥ 8]
5. token invalid/expired/used → 400 AUTH_TOKEN_INVALID
   → web shows "Invalid Token" screen with "request another link" (recoverable)
6. valid → TRANSACTION: DELETE Verification + UPDATE Account.password (argon2)
7. 200 → web shows "Reset Password Success" → link to login
```

---

## 10. Flow — Account email change (three steps)

```
STEP 1 — REQUEST (requires recent re-auth):
  POST /api/v1/auth/change-email { newEmail }
    - credential user: current password must be supplied and verified
    - OAuth-only user: provider re-auth required (PRD §5.11)
    - rate limit 5/h/user
    - newEmail in use → 400 AUTH_EMAIL_IN_USE
    - valid → INSERT Verification (identifier "email-change:{userId}",
      value = pending new email, +1h) → Resend to new address
    - response: "verification sent" — current email REMAINS active

STEP 2 — CONFIRM (link from the new inbox):
  GET /api/v1/auth/change-email?token=...
    - valid → TRANSACTION: UPDATE User.email = newEmail, emailVerified = true
              + DELETE Verification
    - invalid/expired/used → recoverable screen (request new link)
    - success → account email swapped across ALL workspaces (single User row)

STEP 3 — NOTHING: old email stops working immediately after the swap;
  no session invalidation needed (sessions are tied to the user, not the email).
```

**Why this shape:** the old verified email stays authoritative until the new one is proven (domain rule 9) — a user is never left with an unverified, unreachable account.

---

## 11. Edge Cases & Failure Handling

| Case | Behavior |
|---|---|
| Duplicate email at register | 400 `AUTH_EMAIL_IN_USE`, stay on register screen |
| Unverified user logs in | 403 → Verification Pending + resend (rate-limited) |
| Verification/reset token used twice | 400 `AUTH_TOKEN_INVALID` → recoverable screen |
| Resend spam | 429 `AUTH_RATE_LIMITED` + `Retry-After` |
| OAuth cancelled / provider down / no verified email | Redirect to auth screen, retryable message, nothing created |
| Expired session mid-use | API 401 → web redirects to `/login` |
| Login spam / brute force | 10/min/IP, generic credentials error |
| Email change to an in-use address | 400 `AUTH_EMAIL_IN_USE`, current email untouched |
| Email send (Resend) failure | Registration/reset still completes; pending screens offer resend |
| Remember-me | 30d session cookie; same flow, longer expiry |

---

## 12. Dev vs Prod differences

| Concern | Local dev | Production |
|---|---|---|
| Origin | `http://localhost:3000` | `https://shipyard.yonatanem.com` |
| Better Auth `baseURL` / `trustedOrigins` | localhost:3000 | shipyard.yonatanem.com |
| Cookie `Secure` | off | on (Caddy TLS) |
| OAuth callback URLs | `http://localhost:3000/api/v1/auth/callback/...` | prod URL (both registered in provider consoles) |
| Email | Resend dev/test mode (no delivery or sandbox) | Resend prod + verified domain |
| Rate limits | Same (catches abuse early) | Same |
