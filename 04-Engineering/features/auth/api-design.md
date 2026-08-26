# Auth — API Design

**Status:** Draft for implementation (F1)
**Last updated:** 2026-08-26
**Sources:** `features/auth/spec.md` · `features/auth/data-model.md` · `00-architecture.md` §7, §8 · `ADR-001` (Better Auth) · `ADR-003` (Next → API proxy) · `Implementation Plan.md` F1 · Better Auth docs `https://www.better-auth.com/docs/concepts/api` + `https://www.better-auth.com/docs/authentication/email-password` + `https://www.better-auth.com/docs/concepts/email` + `https://www.better-auth.com/docs/concepts/users-accounts` + `better-auth/src/api/index.ts` (baseEndpoints)

> **Principle:** Every HTTP endpoint is provided by Better Auth. We do not write controllers for auth. This doc is the inventory + contract for what Shipyard actually uses, how it is mounted, and the few wrappers/guards we add around it.

---

## 1. Base path & mounting

| Concern | Choice |
|---|---|
| Better Auth `basePath` | `/api/auth` (library default) |
| Express mount (per Implementation Plan F1) | `app.use("/api/v1/auth", toNodeHandler(auth))` — so effective external path is `/api/v1/auth/*` |
| Next.js proxy (ADR-003) | Browser never hits `api:4000` directly. `apps/web` forwards `/api/v1/*` → `http://api:4000/api/v1/*` with cookies forwarded. Caddy exposes only `web:3000`. |
| Auth handler type | `betterAuth({ database: prismaAdapter(prisma, {provider:"postgresql"}), ... })` + `toNodeHandler` (Express). |
| Cookie name | `better-auth.session_token` (HttpOnly, `SameSite=lax`, `Secure` in prod via `useSecureCookies` + `BETTER_AUTH_URL=https`). |
| OpenAPI | Better Auth exposes `ok` (`GET /api/v1/auth/ok` → `{status:"ok"}`) and plugin-generated OpenAPI if `openAPI` plugin added (not for MVP). |

All paths below are given as **effective** paths (`/api/v1/auth/...`).

---

## 2. Endpoint inventory (what Shipyard uses)

All Better Auth endpoints live under the same prefix. Shipyard uses the 13 core endpoints + OAuth callback. No custom endpoints are needed for MVP — every behavior in `spec.md §2–§5` maps to a provided endpoint.

### 2.1 Auth — email/password & session

| # | Method | Effective path | Better Auth `api` | Auth | Spec behavior |
|---|---|---|---|---|---|
| 1 | `POST` | `/api/v1/auth/sign-up/email` | `signUpEmail` | No | §2 Register, §3.1 one-per-email, starts unverified |
| 2 | `POST` | `/api/v1/auth/sign-in/email` | `signInEmail` | No | §3.2 Login (verified only) |
| 3 | `POST` | `/api/v1/auth/sign-out` | `signOut` | Yes (cookie) | §3.3 Logout immediate |
| 4 | `GET` | `/api/v1/auth/get-session` | `getSession` | Cookie | §3.3 Stay logged in, session restore, gate for all authenticated routes |
| 5 | `POST` | `/api/v1/auth/get-session` | `getSession` (alt) | Cookie | Same, POST variant (better-call) |

### 2.2 Email verification

| # | Method | Effective path | `api` | Auth | Spec behavior |
|---|---|---|---|---|---|
| 6 | `GET` | `/api/v1/auth/verify-email?token=...&callbackURL=...` | `verifyEmail` | No | §3.1 Verify link single-use expiring |
| 7 | `POST` | `/api/v1/auth/send-verification-email` | `sendVerificationEmail` | No* | §2 Resend (rate-limited), also triggered on `signIn` when unverified if `sendOnSignIn:true`. Body `{email, callbackURL}` |

*Can be called unauthenticated with email; middleware still checks CSRF/origin.

### 2.3 OAuth

| # | Method | Effective path | `api` | Auth | Spec behavior |
|---|---|---|---|---|---|
| 8 | `POST` | `/api/v1/auth/sign-in/social` | `signInSocial` | No | §2 Sign in with Google/GitHub. Body `{provider:"google"\|"github", callbackURL, scopes?}` → returns `{url, redirect}` |
| 9 | `GET` | `/api/v1/auth/callback/:provider` | `callbackOAuth` | No | §3.2 OAuth callback (state+PKCE validated). On success sets session cookie, on failure redirects with `?error=` — no session created |

`provider` values: `google`, `github` only (spec §6 out of scope: no others).

### 2.4 Password

| # | Method | Effective path | `api` | Auth | Spec behavior |
|---|---|---|---|---|---|
| 10 | `POST` | `/api/v1/auth/request-password-reset` | `requestPasswordReset` | No | §3.4 Forgot → generic 200 (no enumeration). Body `{email, redirectTo}` |
| 11 | `POST` | `/api/v1/auth/reset-password` | `resetPassword` | No (token) | §3.4 Set new password. Body `{token, newPassword}` |
| 12 | `POST` | `/api/v1/auth/change-password` | `changePassword` | Yes | §3.4 Change requires current. Body `{currentPassword, newPassword, revokeOtherSessions?}` |

Aliases: `forget-password` / `forgetPassword` map to the same handler as `request-password-reset` (compat).

### 2.5 Email change

| # | Method | Effective path | `api` | Auth | Spec behavior |
|---|---|---|---|---|---|
| 13 | `POST` | `/api/v1/auth/change-email` | `changeEmail` | Yes | §3.4 Requires recent re-auth + verify new address. Body `{newEmail, callbackURL}` → sends verification to new email; old stays active until verified |

### 2.6 Session utilities (supported but not UI-exposed in MVP)

| # | Method | Effective path | `api` | Auth | Notes |
|---|---|---|---|---|---|
| 14 | `GET` | `/api/v1/auth/list-sessions` | `listSessions` | Yes | Multiple sessions allowed (§3.3). Used internally, no UI in MVP (spec §6). |
| 15 | `POST` | `/api/v1/auth/revoke-session` | `revokeSession` | Yes | Body `{token}` |
| 16 | `POST` | `/api/v1/auth/revoke-other-sessions` | `revokeOtherSessions` | Yes | |
| 17 | `POST` | `/api/v1/auth/revoke-sessions` | `revokeSessions` | Yes | |

Other baseEndpoints exist but are **not used by Shipyard F1**: `updateUser`, `deleteUser` (disabled, spec §6 no deletion), `linkSocialAccount`, `listUserAccounts`, `getAccessToken`/`refreshToken`/`accountInfo` (deferred), `setPassword` (server-only), `ok`/`error` (health).

> **No custom endpoints.** All 5 high-level flows (§4) are covered. We add no `POST /api/v1/auth/check` or similar — `GET /get-session` is the check.

---

## 3. Detailed contracts

All bodies are JSON (`Content-Type: application/json`) except Better Auth also accepts `application/x-www-form-urlencoded` for `sign-in/email` + `sign-up/email` (progressive enhancement). Responses are JSON.

### 3.1 `POST /sign-up/email`

Use: `authClient.signUp.email()` or `auth.api.signUpEmail({body})`.

**Request**
```json
{
  "name": "Alice",
  "email": "alice@example.com",
  "password": "s3curePass!",
  "image": "https://...",
  "callbackURL": "/verify-pending"
}
```
`name,email,password` required. `password` 8–128 (see data-model). `callbackURL` is the redirect after verify link click.

**Success** `200` (when `requireEmailVerification:true` or `autoSignIn:false`, enumeration protection is on — see §7):
```json
{ "user": { "id":"...", "email":"alice@example.com", "name":"Alice", "emailVerified":false, "image":null, "createdAt":"...", "updatedAt":"..." }, "token": null }
```
If enumeration protection off (default without `requireEmailVerification`), success includes `token`/`user`. Shipyard enables `requireEmailVerification:true` so response is always generic on duplicate (see §7).

**Errors** `422`/`400` validation, `429` rate limit. Duplicate when protection on → `200` synthetic user (not an error, see `onExistingUserSignUp` in data-model).

**Spec mapping:** Creates `user` (unverified) + `account` (`providerId:credential`) + `verification` row → `sendVerificationEmail` via Resend.

### 3.2 `POST /sign-in/email`

**Request**
```json
{ "email":"alice@example.com", "password":"...", "rememberMe": true, "callbackURL":"/dashboard" }
```

**Success** `200`
```json
{ "user": { "...": "..." }, "token":"<session token>", "redirect": false, "url": null }
```
Sets `Set-Cookie: better-auth.session_token=...; HttpOnly; SameSite=Lax; Secure; Path=/; Max-Age=604800`

**Errors**
- `401` `INVALID_EMAIL_OR_PASSWORD` — wrong email/password (generic, no leak)
- `403` `EMAIL_NOT_VERIFIED` — unverified (spec §3.2 → verification-pending screen). If `sendOnSignIn:true`, a new verification email is sent as side-effect.
- `429` rate limit
- `400` Zod

### 3.3 `POST /sign-in/social`

**Request**
```json
{ "provider":"google", "callbackURL":"/dashboard", "scopes":["email","profile"], "disableRedirect": false }
```

**Success** `200`
```json
{ "url":"https://accounts.google.com/o/oauth2/v2/auth?state=...&code_challenge=...", "redirect": true }
```
Client redirects to `url`. If `disableRedirect:true`, client handles navigation.

`idToken` branch (native/mobile) returns `{token, user, redirect:false}` — not used in Shipyard web MVP.

### 3.4 `GET /callback/:provider`

Query: `?code=...&state=...&error=...` (OAuth). Better Auth validates `state` + PKCE (`code_verifier` stored in `verification` or cookie per `account.storeStateStrategy`). On success: creates/updates `user`+`account`, creates `session`, sets cookie, redirects `302` to `callbackURL`. On failure (`error`, missing `email`, `email_verified:false` if `requireEmailVerification` per provider): `302` to auth screen with `?error=...`, **no rows created** (spec §3.2, §5.9).

### 3.5 `GET /verify-email` + `POST /send-verification-email`

**Verify:** `GET /verify-email?token=abc&callbackURL=/dashboard`
- Success: `302` to `callbackURL` or `200 {user}` if `autoSignInAfterVerification:true` (also sets session cookie). Deletes `verification` row.
- Errors: `?error=invalid_token` (expired/used) → UI shows recoverable message + resend.

**Resend:** `POST /send-verification-email` `{email, callbackURL:"/"}` → `200` always (generic). Rate-limited (5/min).

### 3.6 `POST /request-password-reset` / `POST /reset-password`

Request: `{email, redirectTo:"https://web/reset-password"}` → `200` always (spec §5.6 generic). Sends email with `url = redirectTo + "?token=..."` if email exists.

Reset: `{token, newPassword}` → `200 {user}`. Token single-use, `resetPasswordTokenExpiresIn:3600`. Optionally `onPasswordReset` hook logs, `revokeSessionsOnPasswordReset:false` for MVP.

### 3.7 `POST /change-password`

Requires `Cookie: better-auth.session_token=...`

Body `{currentPassword, newPassword, revokeOtherSessions?:boolean}` → `200`. Validates current via scrypt. `newPassword` 8–128.

### 3.8 `POST /change-email`

Requires session. Body `{newEmail, callbackURL}` → `200` (generic even if email exists, anti-enumeration). Sends verification to `newEmail`; `user.email` unchanged until verified. If `sendChangeEmailConfirmation` configured, first confirms via current email.

### 3.9 `GET /get-session` / `POST /sign-out`

`get-session`: `GET` with cookie → `200 {session, user} | null` (`null` when no/expired). `Cache-Control: no-store`. Used by `requireSession` middleware for all non-auth routes (Architecture §7.1).

`sign-out`: `POST` with cookie → `200`, `Set-Cookie` cleared, `session` row deleted.

---

## 4. Cookies & headers

| Header | Direction | Notes |
|---|---|---|
| `Cookie: better-auth.session_token=<token>` | Req | Set by Better Auth, forwarded by Next proxy verbatim. |
| `Set-Cookie` | Res | `HttpOnly; SameSite=Lax; Secure (prod); Path=/; Max-Age=604800` (or `0` on sign-out). |
| `Origin` / `Referer` | Req | Validated against `trustedOrigins` + `baseURL`. |
| `X-Forwarded-For` | Req | Used for `ipAddress` + rate limit if `trustProxy` configured. |
| `cache-control: no-store` | Res on `get-session` | Prevents caching session. |

No custom `Authorization: Bearer` — session is cookie-only (spec §3.3).

---

## 5. Rate limiting & security

Configured in `auth.ts` per data-model §7:

- `rateLimit: { storage:"memory", window:60, max:100 }` global.
- Custom: `"/sign-up/email": {window:60, max:5}`, `"/send-verification-email": {window:60, max:5}`, `"/request-password-reset": {window:60, max:5}`, `"/sign-in/email": {window:60, max:10}`.

Enforced by Better Auth middleware before handlers. Returns `429 {code:"RATE_LIMIT_EXCEEDED"}`.

Security: `advanced.useSecureCookies:true`, CSRF via `Origin` + `trustedOrigins=[WEB_URL]`, `SameSite=lax`, scrypt hashing, enumeration protection on sign-up + change-email + reset (generic 200).

---

## 6. Error shape

Better Auth uses `APIError` (via `better-call`). Shipyard's global Express error handler normalizes:

```json
{ "error": { "code": "INVALID_EMAIL_OR_PASSWORD", "message":"Invalid email or password", "details": { "...": "..." } } }
```

Status mapping: `400` Zod/validation, `401` auth, `403` unverified/permission, `404` provider not found, `429` rate limit, `500` unexpected (logged + Sentry, generic to client).

---

## 7. Enumeration protection (important)

When `emailAndPassword.requireEmailVerification:true` (Shipyard: on) or `autoSignIn:false`, Better Auth returns the same `200` for `sign-up/email` whether the email is new or already registered (synthetic user). The `onExistingUserSignUp` hook can notify the existing user. `change-email` and `request-password-reset` are similarly generic. This satisfies spec §5.6 and `data-model.md §5 #6`.

---

## 8. Why no custom endpoints

All spec §2 behaviors map to §2 inventory. The only Shipyard-specific routing is **post-auth decision** (pending invite → onboarding → workspace selection → dashboard) — this is Next.js page logic that calls `GET /get-session` + `GET /api/v1/workspaces` (F2), not an auth endpoint. `POST /sign-up/email` with `autoSignIn:false` + `sendOnSignUp:true` already gives the verified-only flow; `GET /verify-email` with `autoSignInAfterVerification:true` completes it. Password/email change flows are covered. Admin/delete flows are out of scope (§6).

If a future need arises (e.g., `POST /api/v1/auth/verify-password` for re-auth before sensitive settings), it is a Better Auth `verifyPassword` endpoint (already registered, server-only) — not a custom route.

---

## 9. Sequence (register example)

```
Browser → Next /sign-up (form) → POST /api/v1/auth/sign-up/email (via Next proxy, cookie forwarded)
→ Better Auth validates Zod → checks email unique → hashes scrypt → creates user+account+verification (tx)
→ enqueues sendVerificationEmail via Resend → returns 200 generic
→ Browser shows /verify-pending → user clicks email link → GET /verify-email?token=...
→ Better Auth verifies, sets emailVerified=true, deletes verification, optionally sets session cookie
→ 302 to callbackURL → Next reads get-session → routes per F1 "Done when" checklist
```

Same pattern for OAuth (303 via `sign-in/social` → `callback`).

---

## 10. References

- Better Auth API concepts: `https://www.better-auth.com/docs/concepts/api`
- Email & Password auth: `https://www.better-auth.com/docs/authentication/email-password`
- Email verification & reset: `https://www.better-auth.com/docs/concepts/email`
- Users & Accounts (change email): `https://www.better-auth.com/docs/concepts/users-accounts`
- Options ref: `https://www.better-auth.com/docs/reference/options`
- Security: `https://www.better-auth.com/docs/reference/security`
- Base endpoints source: `https://github.com/better-auth/better-auth/blob/main/packages/better-auth/src/api/index.ts`
- Shipyard: `features/auth/spec.md`, `features/auth/data-model.md`, `00-architecture.md` §7–§8, `ADR-003`, `Implementation Plan.md` F1

---

*Next artifact (if needed): `system-design.md` — session middleware, Next proxy details, workspace resolution guard chain. For Auth, data-model + api-design cover F1's technical design per Plan §5 Step 2.*
