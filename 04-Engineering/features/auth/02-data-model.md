# Auth — Data Model

**Module:** `apps/api/src/modules/auth`
**Status:** Draft v0.1 — 2026-08-12
**Stack:** Prisma + PostgreSQL (Neon prod / local container dev) · Better Auth (Prisma adapter)
**PRD source:** §5.1 Authentication

---

## 1. Overview

Auth owns **four tables** — the canonical Better Auth schema (Prisma adapter), hand-written into the Prisma schema with the exact column names Better Auth expects:

| Table | Domain entity | Purpose |
|---|---|---|
| `User` | User | The account — unique identity |
| `Session` | Session | Authentication proof (HttpOnly cookie) |
| `Account` | Account (+ credentials) | OAuth identities **and** the email/password credential |
| `Verification` | VerificationToken | Email verification, password reset, email change tokens |

All four are **hard-delete tables** — no archival semantics (auth state is transient by nature; deleted users remove their sessions/accounts via cascade).

---

## 2. Prisma Schema

```prisma
// ============ AUTH MODULE (Better Auth canonical schema) ============

model User {
  id            String    @id @default(cuid())
  name          String
  email         String    @unique
  emailVerified Boolean   @default(false)
  image         String?
  theme         String    @default("system") // Shipyard: light | dark | system
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  sessions      Session[]
  accounts      Account[]
  // relations to other modules (memberships, created issues, ...) added
  // by their owning modules' docs — never defined here
}

model Session {
  id        String   @id @default(cuid())
  expiresAt DateTime
  token     String   @unique // stored HASHED (Better Auth)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  ipAddress String?
  userAgent String?
  userId    String

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([expiresAt]) // cleanup scans
}

model Account {
  id                    String    @id @default(cuid())
  accountId             String // provider-side user id (OAuth) — or "credential" marker
  providerId            String // "google" | "github" | "credential"
  userId                String
  accessToken           String?
  refreshToken          String?
  idToken               String?
  accessTokenExpiresAt  DateTime?
  refreshTokenExpiresAt DateTime?
  scope                 String?
  password              String? // credential provider: argon2 hash; NULL for OAuth
  createdAt             DateTime  @default(now())
  updatedAt             DateTime  @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([providerId, accountId]) // one provider identity per user
  @@index([userId])
}

model Verification {
  id         String   @id @default(cuid())
  identifier String // e.g. "email-verification:{email}", "reset-password:{token-hash}",
                    // "email-change:{userId}"
  value      String // token (hashed) or the pending new email
  expiresAt  DateTime
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  @@index([identifier])
  @@index([expiresAt])
}
```

---

## 3. Field Notes & Design Rationale

### 3.1 User
- `email` — **unique**; the identity key. Case-normalized (lowercased, trimmed) at validation before insert — prevents `User@x.com` vs `user@x.com` duplicates.
- `emailVerified` — the gate: `false` blocks all protected access for email/password users.
- `name`/`image` — profile basics (Better Auth default); editing flows live in `settings`, values owned here.
- `theme` — **Shipyard addition** (`light | dark | system`), PRD §5.11 Appearance. Account-level, applies across workspaces.
- **No `role` on User** — workspace roles live in `members` (authn ≠ authz).

### 3.2 Session
- `token` — **stored hashed** (Better Auth hashes the cookie token at rest); the plaintext token is only in the HttpOnly cookie.
- `expiresAt` — **decided default: fixed expiry** — 7 days for normal sessions, 30 days when "remember me" is used (Better Auth defaults). Rolling extension considered; rejected for MVP (fixed = simpler invalidation semantics).
- `ipAddress`/`userAgent` — audit context only (no session-management UI in MVP).
- Cascade delete on user deletion; cleanup of expired rows: **lazy deletion** (expired session rejected + row removed on validation) + a periodic sweep scheduled with the future worker container. No cron in MVP.

### 3.3 Account — the dual role table
- `providerId = "credential"` + `password` column → **email/password identity** (argon2 hash, never plaintext).
- `providerId = "google" | "github"` → OAuth identity; `accountId` = provider's user id; tokens/scope stored server-side only.
- `@@unique([providerId, accountId])` — one provider identity can't link to two users; the **merge-by-email** rule lives in the service layer (domain model §3).
- A user can hold multiple rows: credential + google + github — all resolving to the same verified email.

### 3.4 Verification
- `identifier` encodes the token's purpose; `value` holds the (hashed) token or the pending email:
  - `email-verification:{email}` — value = hashed token → verifies the address
  - `reset-password:{token-hash}` — value = hashed token → password reset
  - `email-change:{userId}` — value = **the pending new email** (unverified) → on confirm, User.email + emailVerified updated atomically
- Single-use: the row is **deleted on successful consumption** (an already-used link fails).
- Expiry enforced on read (`expiresAt > now`); expired rows are swept by the same cleanup pass as sessions.

---

## 4. Indexes & Constraints Summary

| Object | Type | Why |
|---|---|---|
| `User.email` | UNIQUE | One identity per email |
| `Session.token` | UNIQUE | Cookie lookup on every request must be O(1) |
| `Session.userId` | INDEX | Load user's sessions |
| `Session.expiresAt` | INDEX | Cleanup sweeps |
| `Account.(providerId, accountId)` | UNIQUE | No duplicate provider links |
| `Account.userId` | INDEX | Load identities for merge-by-email check |
| `Verification.identifier` | INDEX | Token lookup per purpose |
| `Verification.expiresAt` | INDEX | Cleanup sweeps |

**No GIN/tsvector here** — search indexes belong to `issues`/`search` modules.

---

## 5. Data Lifecycle

| Event | What happens |
|---|---|
| Register | INSERT `User` (unverified) + INSERT `Account` (credential + hash) + INSERT `Verification` (email token) |
| Verify email | DELETE `Verification` row → UPDATE `User.emailVerified = true` |
| OAuth login (new) | INSERT `User` (verified) + INSERT `Account` (provider) |
| OAuth login (existing email) | INSERT `Account` only — resolves to existing `User` (no duplicate) |
| Login | INSERT `Session` (token hashed at rest, cookie holds plaintext) |
| Logout | DELETE the `Session` row → cookie invalid immediately |
| Password reset | INSERT `Verification` (reset token) → on confirm: UPDATE `Account.password`, DELETE token |
| Email change | INSERT `Verification` (value = new email) → on confirm: UPDATE `User.email` + `emailVerified` in **one transaction**, DELETE token |
| Session expiry | Rejected on validation; row lazily deleted |
| User deletion (future) | CASCADE removes sessions + accounts (verification rows swept by expiry) |

All multi-step writes (e.g., email change: token consume + user update) run in a **single Prisma transaction** — partial states never persist.

---

## 6. Security Notes

- Passwords: argon2 via Better Auth; never logged, never returned.
- Session tokens hashed at rest; plaintext exists only in the HttpOnly cookie (server-side proxy forwarding only).
- Verification tokens hashed in `value` for reset/verification; email-change stores the plaintext pending email (needed for the confirm step) — the *token* is still the capability, expiry bounds it.
- `Account` provider tokens (access/refresh/id) are secrets: server-side only, excluded from every API response.
- Rate limiting (login/register/resend/reset endpoints) is enforced in middleware — see API design doc.

---

## 7. Sizing & Free-Tier Fit

Auth tables are tiny: ~1KB per user row set (user + sessions + accounts + occasional verification rows). Even 10k users ≈ single-digit MB — **trivially inside Neon's 0.5GB free tier**. No partitioning or special storage config needed.

---

## 8. Decisions Adopted (from domain model open questions)

| # | Question | Decision |
|---|---|---|
| 1 | Session lifetime | **Fixed expiry**: 7 days normal / 30 days remember-me (Better Auth defaults). No rolling extension in MVP |
| 2 | Password policy | Better Auth default: **min 8 chars** (no forced complexity — domain rule §8 keeps it simple; revisit if product feedback asks) |
| 3 | Rate-limit thresholds | Defined in `04-api-design.md` (per endpoint) |
| 4 | Session cap | **Unlimited** multiple sessions (Better Auth default) |

*Flag any of these if you want them changed before implementation.*
