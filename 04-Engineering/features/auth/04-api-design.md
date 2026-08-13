# Auth — API Design

**Module:** `apps/api/src/modules/auth`
**Status:** Draft v0.1 — 2026-08-12
**Base URL:** `/api/v1` (proxied by Next.js; API internal-only)
**Implementation:** Better Auth handler mounted at `/api/v1/auth` on the Express app

---

## 1. Conventions

- **Base path:** all auth routes live under `/api/v1/auth/*` (Better Auth's handler mounted at `/api/v1/auth`).
- **Session transport:** HttpOnly cookie (`better-auth.session_token`) — set by the API, forwarded to the browser by the Next proxy (same-origin: `shipyard.yonatanem.com`). The browser never stores or reads the token in JS.
- **Content:** JSON request/response bodies; forms not used (custom UI calls the JSON endpoints).
- **Success shape:** the resource or `{}`; errors use the global envelope:
  ```json
  { "error": { "code": "AUTH_EMAIL_NOT_VERIFIED", "message": "...", "details": {} } }
  ```
- **Status codes:** `200` success · `201` created (register) · `204` no content (logout) · `400` validation · `401` unauthenticated/expired session · `403` forbidden · `404` not found · `429` rate limited.
- **Contracts:** all request/response Zod schemas live in `packages/shared` (single source of truth, shared with the web app and OpenAPI generation).

---

## 2. Endpoint Map

| Method | Path | Domain op | Source | Auth |
|---|---|---|---|---|
| POST | `/auth/sign-up/email` | register | Better Auth | — |
| GET | `/auth/sign-in/social` | oauthAuthorize (start) | Better Auth | — |
| GET | `/auth/callback/:provider` | oauthAuthorize (callback) | Better Auth | — |
| POST | `/auth/sign-in/email` | login | Better Auth | — |
| GET | `/auth/get-session` | validateSession | Better Auth | session |
| POST | `/auth/sign-out` | logout | Better Auth | session |
| POST | `/auth/send-verification-email` | resendVerification | Better Auth | session |
| GET | `/auth/verify-email` | verifyEmail | Better Auth | token |
| POST | `/auth/request-password-reset` | requestPasswordReset | Better Auth | — |
| POST | `/auth/reset-password` | resetPassword | Better Auth | token |
| POST | `/auth/change-password` | changePassword | Better Auth | session |
| POST | `/auth/change-email` | requestEmailChange | Better Auth | session |
| GET | `/auth/change-email` | confirmEmailChange | Better Auth | token |
| GET | `/auth/error` | OAuth error surface | Better Auth | — |

No custom auth endpoints are needed — the entire domain operation set maps to Better Auth's API. Custom code wraps it: **guards, rate limiting, response shaping, and the session-context middleware** consumed by every other module.

---

## 3. Endpoint Details

### 3.1 Register — `POST /auth/sign-up/email`

**Body:** `{ name: string, email: string, password: string }`
**Validation (Zod):** name 1–64 chars · email valid + normalized (trimmed, lowercased) · password min 8.

**Responses:**
- `201` — `{ user: { id, name, email, emailVerified: false }, token? }` → web shows **Verification Pending** screen (design: `Auth / Signup / Verification Pending`)
- `400` — `AUTH_EMAIL_IN_USE` (email already registered) · `AUTH_INVALID_INPUT` (validation details)
- `429` — `AUTH_RATE_LIMITED` (register limit)

**Behavior:** creates unverified user + credential account + verification token; sends email via Resend; **no protected access** until verified.

### 3.2 Login — `POST /auth/sign-in/email`

**Body:** `{ email, password }` (rememberMe optional)

**Responses:**
- `200` — `{ user, session }` → web redirects: pending invitation → onboarding → sole workspace dashboard → workspace selection (PRD flow)
- `400` — `AUTH_INVALID_CREDENTIALS` (generic — never reveals which field was wrong)
- `403` — `AUTH_EMAIL_NOT_VERIFIED` → web shows Verification Pending (resend allowed, rate-limited)
- `429` — `AUTH_RATE_LIMITED`

**Behavior:** sets the HttpOnly session cookie (7d / 30d remember-me).

### 3.3 OAuth — `GET /auth/sign-in/social?provider=google|github`

- Redirects to the provider's authorization page.
- Callback `GET /auth/callback/:provider` completes the flow server-side.
- **Success:** provider-verified email resolves/creates the user → session cookie set → redirect to invitation/onboarding/workspace (same as login).
- **Failure** (cancel, provider error, no verified email): redirect to the auth screen with a retryable message; **no account/session created**.
- Config: `baseURL` = `https://shipyard.yonatanem.com` (prod) / `http://localhost:3000` (dev); `trustedOrigins` mirrors it; callback URLs registered in the Google/GitHub consoles.

### 3.4 Session — `GET /auth/get-session`

**Responses:**
- `200` — `{ session, user }` (valid)
- `401` — `AUTH_UNAUTHORIZED` (missing/expired) → web redirects to `/login`

**This endpoint is the backbone of the app:** the Next middleware/server components call it through the proxy to gate every protected route; the Express `requireSession` guard (below) uses the same check internally.

### 3.5 Logout — `POST /auth/sign-out`

- `204` — session row deleted, cookie invalidated immediately. No confirmation step (PRD).
- `401` — `AUTH_UNAUTHORIZED` (already logged out — treated as success by the web app).

### 3.6 Verify email — `GET /auth/verify-email?token=...&callbackURL=...`

- Success: token consumed (row deleted), `emailVerified = true`, user signed in → redirect to callbackURL (invitation or onboarding).
- Invalid/expired/used token: redirect to a **recoverable result screen** (design: `Auth / Email Verification / Invalid Token`) with "request a new link" / "go to login" actions.

### 3.7 Resend verification — `POST /auth/send-verification-email`

- `200` — confirmation; destination email shown on the pending screen.
- `429` — `AUTH_RATE_LIMITED` (per-email cooldown). Never creates duplicate accounts.

### 3.8 Password reset

| Step | Endpoint | Notes |
|---|---|---|
| Request | `POST /auth/request-password-reset` `{ email }` | **Generic 200 for all emails** (no account-existence leak, PRD rule). Sends reset link |
| Confirm | `POST /auth/reset-password` `{ newPassword, token }` | Token single-use + expiring; success → login. Invalid/expired → recoverable screen (design: `Auth / Reset Password / Invalid Token`) |
| Change (logged in) | `POST /auth/change-password` `{ currentPassword, newPassword }` | Requires current password (PRD §5.11) |

### 3.9 Email change — `POST /auth/change-email` / `GET /auth/change-email`

- **Request:** `POST /auth/change-email` `{ newEmail }` — requires recent re-auth: current password header for credential users; provider re-auth for OAuth-only users (PRD §5.11).
- **Confirm:** `GET /auth/change-email?token=...` — swaps `User.email` + `emailVerified` **atomically**; the old email remains active until this succeeds.
- Conflicts: new email already in use → `AUTH_EMAIL_IN_USE`.

---

## 4. Session-Context Middleware (consumed by ALL modules)

```ts
// shared kernel — not auth-only, but implemented here
requireSession        // 401 if no valid session → attaches req.user, req.session
requireWorkspaceMember// 403 unless user is a member of the workspace in the route
requireRole(...roles) // 403 unless role ∈ { Owner, Admin, Member } (PRD matrix)
```

Every protected route in every module composes `requireSession` first; workspace scoping uses the **session's userId**, never client input (architecture §7.4).

---

## 5. Rate Limiting (auth endpoints)

| Endpoint | Limit |
|---|---|
| `sign-up/email` | 5/min per IP |
| `sign-in/email` | 10/min per IP |
| `send-verification-email` | 3/hour per email |
| `request-password-reset` | 3/hour per email |
| `change-email` / `change-password` | 5/hour per user |
| OAuth callbacks | 10/min per IP |

Enforced by shared middleware before the handler; `429 AUTH_RATE_LIMITED` with `Retry-After`.

---

## 6. Error Codes (auth domain)

| Code | Status | Meaning |
|---|---|---|
| `AUTH_INVALID_INPUT` | 400 | Zod validation failed (details in `details`) |
| `AUTH_EMAIL_IN_USE` | 400 | Email already registered / target of email change |
| `AUTH_INVALID_CREDENTIALS` | 400 | Wrong email or password (generic) |
| `AUTH_EMAIL_NOT_VERIFIED` | 403 | Verified email required for access |
| `AUTH_UNAUTHORIZED` | 401 | Missing/expired/invalid session |
| `AUTH_TOKEN_INVALID` | 400 | Verification/reset token invalid, expired, or used |
| `AUTH_RATE_LIMITED` | 429 | Rate limit hit (`Retry-After` header) |
| `AUTH_OAUTH_FAILED` | 400 | Provider error / no verified email (retryable) |

---

## 7. Web Integration (consumers)

| Web route (custom UI from design repo) | Calls |
|---|---|
| `/register` (`Auth / Signup`) | `sign-up/email` |
| `/verify-email` (`Verification Pending`) | `send-verification-email` |
| `/verify-email/result` (`Success / Invalid Token`) | `verify-email` (link) |
| `/login` (`Auth / Login`) | `sign-in/email`, `sign-in/social` |
| `/forgot-password` (+ Email Sent, Invalid Token, Success screens) | `request-password-reset`, `reset-password` |
| App shell | `get-session` (server-side via proxy) |
| User menu | `sign-out`, `change-password`, `change-email` |

All calls go **through the Next proxy** (`/api/*` rewrite → API internal) — the browser never hits the API origin directly (ADR-003).

---

## 8. OpenAPI & Shared Contracts

- Every schema above is a Zod schema in `packages/shared` → `openapi` type generation documents `/api/v1/auth/*` automatically (post-MVP: publish the OpenAPI spec as public docs).
- Better Auth's own types are mapped to shared DTOs at the module boundary; other modules never import Better Auth types directly.

---

## 9. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | OAuth callback URLs: single prod + dev localhost — any staging URL needed? | Prod + local only (per environments decision) |
| 2 | Remember-me surfaced as a checkbox on `/login`? | Design repo login screen — confirm placement |
| 3 | Cookie name/attributes beyond Better Auth defaults (`Secure`, `SameSite=Lax`)? | Defaults sufficient; CSP story handled at Next layer |
