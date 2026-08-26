# Auth — Data Model

**Status:** Draft for implementation (F1)
**Last updated:** 2026-08-26
**Sources:** `features/auth/spec.md` · `00-architecture.md` §8, §9 · `ADR-001` (Better Auth + Prisma + Postgres) · Better Auth docs `https://www.better-auth.com/docs/concepts/database` (v1.7) + `https://www.better-auth.com/docs/adapters/prisma` + `https://www.better-auth.com/docs/reference/options`
**Owner:** `apps/api` — Better Auth is the source of truth for this module's tables. This doc explains what it creates and why, it does not redesign it.

---

## 1. Overview

Auth owns **identity tables only**. Authorization (roles, membership, workspace access) lives in `members`. Better Auth provides the entire persistence layer for F1 — 4 core tables, their columns, indexes, and relations. We pin to its conventions exactly so `npx @better-auth/cli generate` is authoritative.

No hand-rolled `users` table, no custom table/column renaming, no `additionalFields` in MVP. If a future feature needs a claim on the session/user (e.g. `role`), it is added via `user.additionalFields` + re-generate, never by hand-editing Prisma.

**Storage for MVP:** PostgreSQL (Neon prod, local Postgres dev) via `prismaAdapter(prisma, { provider: "postgresql" })`. Sessions in DB (no `secondaryStorage` / Redis yet), `cookieCache` disabled — DB is the truth. `advanced.database.joins: true` after generation so `getSession` can join.

---

## 2. Core schema (Better Auth owned)

### 2.1 `user` — `@@map("user")`

One row per human. Email is the identity key.

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK | Better Auth base62 (default `generateId`). Never `serial`/`uuid` for MVP. |
| `name` | `String` | — | Required by Better Auth signup. Display name. |
| `email` | `String` | `@@unique([email])` | Case-sensitive uniqueness enforced in DB. 1 account per email (spec §5.1). |
| `emailVerified` | `Boolean` | `@default(false)` | Gate for §5.2. OAuth providers set `true` on create. |
| `image` | `String?` | — | Avatar URL from OAuth or later R2. Nullable. |
| `createdAt` | `DateTime` | `@default(now())` | |
| `updatedAt` | `DateTime` | `@updatedAt` | |

> No `password` here — credential hash lives on `account.password`.

### 2.2 `session` — `@@map("session")`

Supports multi-device (spec §3.3). Cookie `better-auth.session_token` is `HttpOnly, SameSite=lax, Secure (prod)`.

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK | |
| `userId` | `String` | FK → `user.id` `onDelete: Cascade` + `@@index([userId])` | |
| `token` | `String` | `@@unique` | Opaque session token (in cookie). |
| `expiresAt` | `DateTime` | — | `session.expiresIn = 604800` (7d). Refreshed when `updateAge` (1d) reached. |
| `ipAddress` | `String?` | — | From `advanced.ipAddress` tracking, for audit/rate-limit. |
| `userAgent` | `String?` | — | |
| `createdAt` | `DateTime` | `@default(now())` | |
| `updatedAt` | `DateTime` | `@updatedAt` | `updateAge` sliding extension. |

Logout = delete row immediately. Expired sessions redirect to login (§5.5).

### 2.3 `account` — `@@map("account")`

One row per auth method linked to a user. Better Auth identifies a provider identity by the **compound** `(issuer, accountId)` — `id` is the local row PK used by APIs.

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK | Local row id. Use this when API asks for `accountId`. |
| `userId` | `String` | FK → `user.id` `Cascade` + `@@index([userId])` | |
| `accountId` | `String` | — | Stable id within issuer. For `credential` = `user.id`; for OAuth = provider's `sub`. |
| `providerId` | `String` | — | `credential` \| `google` \| `github` (MVP). |
| `issuer` | `String` | — | Authority that issued `accountId`. `local:credential` for password, `local:oauth:<provider>` (percent-encoded) for OAuth, or real OIDC issuer. Adapter enforces `UNIQUE(issuer, accountId)`. |
| `accessToken` | `String?` | `@db.Text` | OAuth, encrypted if `account.encryptOAuthTokens:true`. |
| `refreshToken` | `String?` | `@db.Text` | |
| `idToken` | `String?` | `@db.Text` | |
| `accessTokenExpiresAt` | `DateTime?` | — | |
| `refreshTokenExpiresAt` | `DateTime?` | — | |
| `scope` | `String?` | — | |
| `password` | `String?` | `@db.Text` | scrypt hash (spec §3.1). Only for `providerId=credential`. |
| `createdAt` | `DateTime` | `@default(now())` | |
| `updatedAt` | `DateTime` | `@updatedAt` | |

Identity resolution (spec §5.3): OAuth sign-in with an email that already exists links to that `user` via `accountLinking.enabled:true, trustedProviders:[google,github]`. No duplicate users, no merge UI. Failure (cancel, missing `email_verified`) creates nothing (§3.2, §5.9).

### 2.4 `verification` — `@@map("verification")`

Single-use, expiring tokens. One table for all flows (spec §5.4): email verification, password reset, email-change confirmation. With `secondaryStorage` absent, rows live here; with Redis later they'd move but schema stays.

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK | |
| `identifier` | `String` | `@@index([identifier])` | Email or `email-change:<userId>` depending on flow. |
| `value` | `String` | `@db.Text` | Token (hashed if `verification.storeIdentifier:"hashed"` — default plain for MVP). |
| `expiresAt` | `DateTime` | — | `emailVerification.expiresIn:3600`, `resetPasswordTokenExpiresIn:3600`, change-email similar. |
| `createdAt` | `DateTime` | `@default(now())` | |
| `updatedAt` | `DateTime` | `@updatedAt` | Cleanup removes expired on fetch (`disableCleanup:false`). |

Tokens are deleted after first successful use. Resend is rate-limited (spec §5.4) via `rateLimit` (memory for MVP, not a table).

---

## 3. Prisma schema (canonical — paste-ready)

Generated by `npx @better-auth/cli generate` with `prismaAdapter` + `provider:"postgresql"`. Keep this exact shape; CLI is authoritative if docs drift.

```prisma
// apps/api/prisma/schema.prisma — F1 Auth slice
// datasource + generator already in file, this is appended by the CLI

model User {
  id            String    @id
  name          String
  email         String
  emailVerified Boolean   @default(false)
  image         String?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  sessions      Session[]
  accounts      Account[]

  @@unique([email])
  @@map("user")
}

model Session {
  id        String   @id
  expiresAt DateTime
  token     String   @unique
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  ipAddress String?
  userAgent String?
  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@map("session")
}

model Account {
  id                    String    @id
  accountId             String
  providerId            String
  userId                String
  user                  User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  issuer                String
  accessToken           String?   @db.Text
  refreshToken          String?   @db.Text
  idToken               String?   @db.Text
  accessTokenExpiresAt  DateTime?
  refreshTokenExpiresAt DateTime?
  scope                 String?
  password              String?   @db.Text
  createdAt             DateTime  @default(now())
  updatedAt             DateTime  @updatedAt

  @@index([userId])
  @@map("account")
}

model Verification {
  id         String   @id
  identifier String
  value      String   @db.Text
  expiresAt  DateTime
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  @@index([identifier])
  @@map("verification")
}
```

**Notes:**
- `@@map` keeps Prisma PascalCase while DB stays lowercase as Better Auth expects.
- `@db.Text` for tokens/hashes prevents varchar limits.
- No `@@unique([issuer, accountId])` in Prisma output — constraint is enforced by the adapter at runtime (do not add manually unless you own the DB trigger).
- Relations required for `advanced.database.joins:true`.

### ER (text)

```
User 1──∞ Session (userId FK, cascade)
User 1──∞ Account (userId FK, cascade)
# Verification is standalone, keyed by identifier (email)
```

---

## 4. Indexes & constraints summary

| Constraint | Where | Purpose |
|---|---|---|
| `@@unique([email])` on `user` | DB | Spec §5.1, §2 one-per-email. API returns generic error to avoid enumeration (§5.6). |
| `@@unique(token)` on `session` | DB | Session lookup hot path. |
| `UNIQUE(issuer, accountId)` | Adapter | Prevents duplicate provider accounts. |
| `@@index([userId])` on `session`/`account` | DB | `getSession`, list/revoke flows. |
| `@@index([identifier])` on `verification` | DB | Token lookup + expired cleanup. |
| `onDelete: Cascade` | FK | Deleting user (future admin) removes sessions/accounts atomically. |

Password is never `@@unique`; verification `value` is never indexed (token lookup uses `identifier`).

---

## 5. Mapping business rules → storage

| Spec §5 | Rule | Enforced where |
|---|---|---|
| 1 | Emails unique, user may be in N workspaces | DB unique + `members` feature owns workspace link, not here. |
| 2 | Email/password needs `emailVerified` | `user.emailVerified` + `emailAndPassword.requireEmailVerification:true` + middleware returns 401 → verification-pending screen. |
| 3 | Provider-verified satisfies, no duplicates | `accountLinking.enabled` + `issuer/accountId` uniqueness. |
| 4 | Tokens single-use expiring + rate-limit resend | `verification.expiresAt` + delete on use + `rateLimit` on `/send-verification-email`. |
| 5 | Logout invalidates immediately; expired→login | `DELETE FROM session WHERE token` + `expiresAt` check in session middleware. |
| 6 | Reset response generic | Service returns 200 regardless; relies on §1 unique. |
| 7 | Email change needs re-auth + verify new | `user.changeEmail.enabled:true` + Better Auth change-email flow uses `verification` (old stays active until verified). |
| 8 | Password change needs current | `account.password` verify via Better Auth `updatePassword`. |
| 9 | Failed flows leave nothing | Better Auth transaction: user+account+verification created atomically or not at all. |
| 10 | Auth ≠ authz | No `role`/`workspaceId` columns here. |

---

## 6. Lifecycles & token handling

- **Register:** `user` (unverified) + `account` (credential, scrypt hash) + `verification` row → email link. Link click → verify → `emailVerified=true` → delete verification. Expired/used link shows recoverable message + resend.
- **OAuth:** `GET /api/auth/sign-in/social` → state in `verification` (or cookie if `storeStateStrategy:"cookie"`), callback validates `state+pkce` → find or create `user` (verified) + upsert `account` → create `session`. Cancel/error → no rows.
- **Session:** created on login/verify/OAuth, cookie `better-auth.session_token`. Refresh slides `expiresAt` when `now - updatedAt > updateAge`. Multiple rows per user allowed; no cap (spec §7.3).
- **Password reset:** `POST /request-password-reset` → generic 200 + `verification` if email exists → email link → `POST /reset-password` consumes token → update `account.password` → delete token. `revokeSessionsOnPasswordReset:false` for MVP.
- **Email change:** re-auth check → `verification` for new email → click link → swap `user.email`. Resend rate-limited.
- **Cleanup:** expired `verification` rows purged on next fetch; optional cron `DELETE WHERE expiresAt < now()` post-MVP.

Email sending via `Resend` adapter in `emailVerification.sendVerificationEmail` / `emailAndPassword.sendResetPassword` / `user.changeEmail.sendChangeEmailConfirmation`. In dev, log to console if `RESEND_API_KEY` absent.

---

## 7. Configuration decisions (aligns with Better Auth options ref)

```ts
// apps/api/src/lib/auth.ts (illustrative, not this doc's code)
betterAuth({
  database: prismaAdapter(prisma, { provider: "postgresql" }),
  baseURL: process.env.BETTER_AUTH_URL,
  secret: process.env.BETTER_AUTH_SECRET,
  basePath: "/api/auth", // mounted as /api/v1/auth in Express
  trustedOrigins: [process.env.WEB_URL!],
  emailAndPassword: {
    enabled: true,
    requireEmailVerification: true,
    minPasswordLength: 8, maxPasswordLength: 128,
    autoSignIn: false, // force verify-first flow (spec §3.1)
    sendResetPassword: async ({ user, url }) => { /* Resend */ },
    resetPasswordTokenExpiresIn: 3600,
  },
  emailVerification: {
    sendOnSignUp: true,
    autoSignInAfterVerification: true,
    expiresIn: 3600,
    sendVerificationEmail: async ({ user, url }) => { /* Resend */ },
  },
  socialProviders: {
    google: { clientId: ..., clientSecret: ..., scope: ["email","profile"] },
    github: { clientId: ..., clientSecret: ..., scope: ["user:email"] },
  },
  account: { accountLinking: { enabled: true, trustedProviders: ["google","github"] } },
  user: { changeEmail: { enabled: true }, deleteUser: { enabled: false } },
  session: { expiresIn: 604800, updateAge: 86400, cookieCache: { enabled: false } },
  advanced: { database: { generateId: undefined, joins: true }, useSecureCookies: true },
  rateLimit: { enabled: true, storage: "memory", window: 60, max: 100, customRules: { "/sign-up/email": { window: 60, max: 5 } } },
})
```

- **IDs:** default Better Auth (base62). No `serial`/`uuid`/`generateId:false` — lets library own it.
- **Cookies:** `HttpOnly, SameSite=lax, Secure` in prod (`useSecureCookies` via `BETTER_AUTH_URL=https`). No `crossSubDomainCookies`.
- **Rate limit:** memory for MVP, 5/min on sign-up/resend to satisfy spec §5.4.
- **No plugins:** `twoFactor/passkey/organization` deferred (spec §6).

---

## 8. Migration & generation workflow

```bash
pnpm add better-auth @better-auth/prisma-adapter
# auth.ts as above, with prisma from apps/api/src/lib/prisma.ts (PrismaPg adapter)
npx @better-auth/cli generate   # merges models into apps/api/prisma/schema.prisma
pnpm --filter @shipyard/api db:migrate -- --name add-better-auth  # creates prisma/migrations/xxx_add_better_auth
pnpm --filter @shipyard/api db:generate  # regenerates src/generated
```

- Never hand-edit the generated migration after `prisma migrate dev`.
- Re-run `generate` after any `user.additionalFields` / plugin change.
- CI deploys with `prisma migrate deploy` (per `00-architecture` §9).

Local dev: `docker-compose` Postgres or Neon branch; `DATABASE_URL` from `apps/api/.env` + root `.env` via `prisma.config.ts`.

---

## 9. What we intentionally do NOT model

| Deferred | Why |
|---|---|
| `user.role`, `workspaceId`, `members` | Owned by `members` feature. |
| `passkey`, `twoFactor`, `sso` tables | Spec §6 out of scope. |
| `rateLimit` table | `storage:"memory"` for MVP. |
| `secondaryStorage` (Redis) | No worker/queue in MVP (`00-architecture` §11). |
| Avatar `image` as R2 key | Store URL string for MVP; R2 signed URLs handled in `settings` (F11). |
| Account deletion flow | `user.deleteUser.enabled:false` (spec §6). |

---

## 10. Open product questions — resolved at data layer

| Spec §7 | Decision |
|---|---|
| 1 Session lifetime | 7d fixed + 1d sliding. Not UX-visible. |
| 2 Password policy | 8–128 chars, scrypt via Better Auth. Stronger policy later via `password.hash/verify` custom. |
| 3 Session cap | None — multiple sessions allowed, indexed by `userId`. |

---

## 11. References

- Better Auth Database core schema: `https://www.better-auth.com/docs/concepts/database#core-schema`
- Prisma adapter: `https://www.better-auth.com/docs/adapters/prisma`
- Options ref (emailVerification, emailAndPassword, session, account, rateLimit, advanced): `https://www.better-auth.com/docs/reference/options`
- Security (hashing, cookies, CSRF, rate limiting, enumeration protection): `https://www.better-auth.com/docs/reference/security`
- Shipyard architecture: `00-architecture.md`, `ADR-001-stack.md`, `Implementation Plan.md` F1

---

*Next artifact: `api-design.md` — Better Auth endpoints mounted at `/api/v1/auth` (all provided, plus any custom wrapper) and Next proxy behavior per ADR-003.*
