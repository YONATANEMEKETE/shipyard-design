# Settings — Data Model

**Status:** Draft for review
**Last updated:** 2026-09-04
**Sources:** `features/settings/spec.md` · `features/auth/data-model.md` (F1 precedent — Better Auth `user` table: `name`, `image`, email-owned-by-Auth, R2-URL-for-MVP note §9) · `features/projects/data-model.md` (F4 precedent — `view_preference` table + scopes, absent ⇒ `LIST`) · `features/issues/data-model.md` (F5 precedent — `ViewScope += ISSUE` reuse) · `00-architecture.md` §5, §8 (uploads through API → R2; config validated at boot) · `ADR-001` (Prisma + Postgres) · `ADR-002` (shared contracts) · `ADR-004` (R2 object storage) · `Implementation Plan.md` F11
**Owner:** `apps/api` — Prisma-owned (hand-modeled; one new table, zero edits to Better Auth tables).

> **Locked scope (2026-09-04):** `user_preference` table (not a `user` column); public R2 bucket + unguessable keys; jpeg/png/webp ≤2MB, no server crop; replace-then-clean / clear-plus-cleanup avatar semantics; default theme `SYSTEM`; default view `LIST`; display name trim 1–100; session-only settings routes with view toggles staying on F4 workspace routes.

---

## 1. Overview

Settings owns the **account surface**: display name, avatar, and theme — plus the settings-shell UX over domains that own their rules (security → Auth, workspace/members → Workspace/Members, view toggles → the existing `view_preference` endpoints). It is deliberately the thinnest milestone: one new table, no edits to any existing table, no duplicated business logic.

| Table / Change | Purpose | Formalized by |
|---|---|---|
| `user_preference` | Account-wide prefs: theme today (`LIGHT\|DARK\|SYSTEM`, default `SYSTEM`); home for future prefs | **F11 (this milestone)** |
| `user.image` | Avatar URL — **written, never migrated** (column exists since F1) | F1 owns; F11 writes via R2 flow (§6.3) |
| `user.name` | Display name — **written, never migrated** | F1 owns; F11 writes bounded (§6.1) |
| `view_preference` | View toggles — **reused, untouched** (F4 table, F4/F5 endpoints) | F4 owns; F11 only links |

---

## 2. Core schema (Prisma-owned)

### 2.1 `user_preference` — the only new table

One row per user, created lazily on first preference write. Account-wide by construction (no `workspaceId` — theme follows the human, spec rule 2).

| Column | Type | Attr | Notes |
|---|---|---|---|
| `userId` | `String` | PK, FK → `user.id` `onDelete: Cascade` | One row per account. User delete removes prefs (private data, same reasoning as notification inbox). |
| `theme` | `ThemePreference` | `@default(SYSTEM)` | `LIGHT \| DARK \| SYSTEM`. Absent row reads as `SYSTEM` (same absent-means-default discipline as view prefs). |
| `createdAt` | `DateTime` | `@default(now())` | |
| `updatedAt` | `DateTime` | `@updatedAt` | Theme flips touch this; no other writer exists. |

```prisma
enum ThemePreference {
  LIGHT
  DARK
  SYSTEM
}

model UserPreference {
  userId    String          @id
  theme     ThemePreference @default(SYSTEM)
  createdAt DateTime        @default(now())
  updatedAt DateTime        @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("user_preference")
}
```

`User` gains `preference UserPreference?`. Nothing else changes in any Prisma model — in particular **no column is added to Better Auth's `user`** (hand-edits around generated auth tables risk upgrade drift; the preference table keeps F11's footprint isolated and gives future prefs like locale an obvious home).

### 2.2 Avatar storage — R2 convention (no migration)

Avatars live in Cloudflare R2 under a public bucket with unguessable keys; the only persisted pointer is the existing `user.image` URL (auth data-model §9: "store URL string for MVP").

| Concern | Convention |
|---|---|
| Bucket | Public-read R2 bucket (env `R2_AVATAR_BUCKET`); random keys are the access control — no signed-URL minting per render (every card contract since F5 carries plain `image` URLs; per-render minting would touch all read paths for no MVP threat model) |
| Key scheme | `avatars/:userId/:uuid.:ext` — `uuid` is 128-bit random (`crypto.randomUUID()`), `ext` derived from the validated MIME (`jpg`/`png`/`webp`) |
| Upload | Through the API only (arch §8): validate type/size → `PutObject` with `ContentType` + `Cache-Control: public, max-age=31536000, immutable` (new key per upload ⇒ immutable caching is safe) → persist URL in `user.image` |
| Replace | Upload new key → persist new URL → best-effort delete old object (R2 failure never fails the request — the DB URL is source of truth; orphaned objects are inert and listed for periodic cleanup post-MVP) |
| Remove | Set `user.image = NULL` + best-effort object delete (spec Q1: delete = clear) |
| Config | Endpoint/keys/bucket from server env, validated at boot (fail fast per arch §8); tests inject an in-memory fake adapter (§8) |

### 2.3 Reused, untouched: `view_preference`, `user.name`, `user.image`, Auth identity

- View toggles (`PROJECT`/`ISSUE` scopes) keep their F4 endpoints byte-for-byte — F11 adds no scope (cycles deliberately has none) and no route.
- `user.name` / `user.image` gain writers, not columns. Email, password, verification, OAuth, sessions stay Auth-exclusive: the settings profile schema is `.strict()` so an `email` key is a `400`, proving rule 4 structurally (§4).

---

## 3. Key decisions & alternatives

### D1 — `user_preference` table, not a `user` column (locked)

**Decision:** §2.1 as specified. *Rejected:* `user.theme` — couples F11 to Better Auth's generated table (migration-order coupling on every auth-package upgrade) for a single enum that will gain siblings (locale is the obvious next row-field, not the next table).

### D2 — Public bucket + unguessable keys, not private + signed URLs (locked)

**Decision:** §2.2 as specified. Avatars are identity furniture rendered on nearly every card in the product — they are not secrets, and unguessable 128-bit keys in a dedicated prefix are the standard MVP posture. *Rejected:* private bucket with per-render signed minting — would thread URL-minting through members/issues/comments/cards (all plain-`image` contracts) and add a latency hop to every list for a privacy property profile pictures don't need. Upload/delete stay strictly self-only server-side — that is the real authorization boundary, enforced where the write happens.

### D3 — Avatar bounds: jpeg/png/webp ≤2MB, no GIF, no server processing (locked)

**Decision:** allowlist `image/jpeg`, `image/png`, `image/webp` (MIME sniffed, extension matched), size ≤2MB enforced before buffering, no crop/resize/re-encode — CSS framing handles presentation, keeping upload a three-step path (validate → put → store). *Rejected:* GIF (animated-avatar scope: playback, frame extraction, larger abuse surface — out of MVP); larger caps (2MB comfortably holds a 1024px avatar in all three formats); dimension validation (adds an image-decode dependency for a property CSS already absorbs).

### D4 — Replace-then-clean / clear-plus-cleanup, DB-URL-wins (locked)

**Decision:** §2.2 replace/remove rows as specified — persist-then-cleanup ordering means a crashed cleanup leaves an orphaned object (inert, unreferenced), never a broken image. R2 errors during cleanup are logged, never surfaced. *Rejected:* delete-old-first (crash window = user with no avatar and a wasted upload); surfacing R2 cleanup failures (turns storage hygiene into user-facing errors).

### D5 — Strict profile schema: `email` is a 400, not a silent strip (locked)

**Decision:** `updateProfileSchema = z.object({ name }).strict()` — an `email` key fails validation instead of being ignored, so "Settings never writes email" (rule 4) is test-observable rather than conventional. Same posture as cycles' no-`status`-field (unrepresentable beats unenforced).

### D6 — Theme default `SYSTEM`, view default `LIST`, detection client-side (locked, spec Q2/Q3)

**Decision:** absent `user_preference` row ⇒ `SYSTEM` (OS detection runs client-side; the server stores the choice only). Absent `view_preference` row ⇒ `LIST` (reaffirmed F4 contract, unchanged). No migration seeds rows — defaults are read-time, not stored.

### D7 — Display name trim 1–100 (locked)

**Decision:** `z.string().trim().min(1).max(100)` — first bound ever set on `user.name` (F1 left it to Better Auth). 100 chars covers long names without breaking card layouts; empty-after-trim rejected (nameless accounts break mention rendering, which keys off `user.name` words).

---

## 4. Shared contracts (`packages/shared`)

Added in F11, consumed by `api` and `web` (ADR-002). View-preference contracts stay where F4/F5 defined them — re-exported here for the settings shell's convenience only.

```ts
// zod enum — mirrors Prisma enum §2.1
export const themePreferenceSchema = z.enum(["LIGHT", "DARK", "SYSTEM"]);

// canonical bounds (D6/D7)
export const displayNameSchema = z.string().trim().min(1).max(100);
export const avatarMimeAllowlist = ["image/jpeg", "image/png", "image/webp"] as const;
export const AVATAR_MAX_BYTES = 2 * 1024 * 1024; // 2MB (D3)

// request contracts owned by the settings module
export const updateProfileSchema = z.object({
  name: displayNameSchema,
}).strict(); // D5 — email key (or anything else) is a 400, never a silent strip

export const setAppearanceSchema = z.object({
  theme: themePreferenceSchema,
});

// response contracts
export const profileCardSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email(), // read-only here — Auth owns writes (rule 4)
  image: z.string().nullable(),
  emailVerified: z.boolean(),
});

export const appearanceSchema = z.object({
  theme: themePreferenceSchema, // SYSTEM when no row exists (D6)
});

export const avatarCardSchema = z.object({
  image: z.string(), // public R2 URL just persisted
});
```

Avatar upload has no JSON body schema — `multipart/form-data` with a single `avatar` file field, validated in the controller against `avatarMimeAllowlist` + `AVATAR_MAX_BYTES` before any R2 call (api-design §5.1).

---

## 5. Integrity invariants → spec rule mapping

| Spec rule | Enforcement point |
|---|---|
| 1 — profile account-wide | `user.name`/`user.image`/`user_preference` keyed by `userId` only — no workspace scoping exists to vary by |
| 2 — theme account-wide; views per (user, workspace, kind) | `user_preference` PK `userId` vs `view_preference` PK `(workspaceId, userId, scope)` — scopes differ structurally, not conventionally |
| 3 — avatar type/size validated; authorized serving | Controller allowlist + pre-buffer size cap (D3); public-keys posture + self-only writes (D2); R2 config boot-validated |
| 4 — email/Auth never written here | `.strict()` profile schema (D5) + zero code path touching Auth tables |
| 5 — workspace/member writes via owning services only | No such code exists in this module — settings shell routes to owning screens; F11 adds no proxy routes |
| 6 — omit, don't disable | Web-shell concern: permission-derived navigation built from membership role (api-design §9.3) |
| 7 — independent view prefs; subviews never overwrite | Unchanged F4/F5 behavior — absent ⇒ `LIST`, list-only subviews never `PUT` (reaffirmed, not reimplemented) |

Integrity summary — constraints added in F11:

| Constraint | Where | Purpose |
|---|---|---|
| PK `userId` | `user_preference` | One pref row per account |
| FK `user_preference.userId → user` `Cascade` | `user_preference` | Prefs die with the account (private data) |
| (R2 key uniqueness) | `avatars/:userId/:uuid` convention | 128-bit random keys — no DB constraint needed or wanted |

---

## 6. Lifecycle semantics at the data layer

All writes are single-row, single-statement account operations — no multi-write transactions except avatar replace (persist + best-effort cleanup, explicitly non-atomic by design per D4).

### 6.1 Profile update (spec §3.1/§4.1)

```text
PATCH { name } → trim → validate 1–100 → UPDATE "user" SET name WHERE id = session.userId
→ 200 profileCard (email/image echoed read-only)
```

`email` (or any extra key) in the body ⇒ `400 VALIDATION_ERROR` via `.strict()` (D5). No workspace membership required — zero-workspace users in onboarding can set their name (session-only route, api-design §4). Applies everywhere instantly (every card joins `user.name` live).

### 6.2 Appearance get/set (spec §3.2/§4.2)

```text
GET → SELECT theme WHERE userId → row ? { theme } : { theme: "SYSTEM" }   // default read, D6
PUT { theme } → UPSERT user_preference (userId) SET theme → 200 { theme }
```

Upsert is the only write (lazy row creation — no backfill, no seed). Invalid enum ⇒ `400`. Client applies instantly + persists server-side (spec §4.2); OS-following for `SYSTEM` runs purely client-side.

### 6.3 Avatar upload/replace/remove (spec §3.1/§4.1, R2 §2.2)

```text
POST multipart [avatar] →
  assert MIME ∈ allowlist + ext matches + bytes ≤ 2MB (else 400, nothing stored)
  key = avatars/:userId/:uuid.:ext → R2 PutObject (public, immutable cache headers)
  UPDATE "user" SET image = url WHERE id → 200 { image: url }
  best-effort: R2 delete OLD key (failure logged, never surfaced — D4)

DELETE { confirm: true } →
  UPDATE "user" SET image = NULL → best-effort R2 delete old key → 200 profileCard (image: null)
```

OAuth-supplied `image` URLs are overwritten identically (no special-casing — replace is replace). Missing file field / empty part ⇒ `400`. Request-body size cap sits below the 2MB file cap at the middleware layer so oversized floods die before buffering.

### 6.4 View toggles (no new semantics — reuse map)

`GET/PUT .../view-preferences/:scope` (F4 #9/#10, `PROJECT`/`ISSUE`) serve the settings-adjacent toggles on owning pages exactly as today. F11 writes no preference code and widens no enum. Cycles has no toggle by F7 design — nothing to persist.

### 6.5 Delegation (no-op by design — the point of §3.4)

Security (password/email), workspace details/members/danger-zone: the settings shell deep-links to Auth/Workspace/Members screens. There are deliberately no proxy `POST /settings/change-password`-style routes — a proxy would duplicate guards it cannot own (rule 5). The api-design documents the link map, not endpoints.

### 6.6 Archived-workspace interaction

None — profile/appearance/avatar/view-toggle writes are account-scoped or already-gated upstream. An archived workspace changes nothing about your name, theme, or avatar; view toggles in a frozen workspace remain harmless per-user metadata (F4 matrix reaffirmed).

---

## 7. Forward handoffs

| Consumer | Contract | Landed |
|---|---|---|
| **Auth (F1)** | Settings writes `user.name`/`user.image` only — sessions, passwords, emails, OAuth untouched (rule 4) | **F11 implements** (bounded writers) |
| **Projects/Issues (F4/F5)** | View-preference endpoints reused unchanged; no new scope | — (reuse, zero code) |
| **Future prefs (locale, etc.)** | `user_preference` gains nullable columns (each defaulted) — no new table per pref | — (future) |
| **R2 periodic cleanup** | Orphaned avatar objects (D4 crash windows) listed for a post-MVP janitor — no table tracks them | — (future) |

---

## 8. Migration workflow

Hand-modeled Prisma (like all prior features):

```bash
# 1 — add ThemePreference enum + UserPreference model + User.preference back-relation
#     (no edits to user/session/account/verification models)
# 2 — R2 bucket created out-of-band (dashboard/provider console, NOT a migration):
#     public-read bucket + env R2_AVATAR_BUCKET (+ endpoint/keys); boot-validated
# 3 — run
pnpm --filter @shipyard/api db:migrate -- --name add_settings_user_preference
pnpm --filter @shipyard/api db:generate
```

- The migration produces: 1 table (`user_preference`), 1 enum (`ThemePreference`), FK + back-relation. No raw-SQL appends — Prisma expresses everything F11 needs. No backfill/seed — defaults read live (D6).
- Tests inject an in-memory fake R2 adapter (map of key→bytes); no emulator, no network in unit/integration tests. Local dev uses the real free-tier bucket.
- The F1 Testcontainers harness applies migrations automatically each test run.

**Post-migration verification (manual, once):**

```sql
-- one pref row per user at most (PK guarantees; confirms no legacy drift)
SELECT user_id, count(*) FROM user_preference GROUP BY 1 HAVING count(*)>1;
-- every pref row points at a live user (FK guarantees; sanity)
SELECT p.user_id FROM user_preference p LEFT JOIN "user" u ON u.id = p.user_id WHERE u.id IS NULL;
-- theme values within enum (check-constraint equivalent eyeball)
SELECT theme, count(*) FROM user_preference GROUP BY 1;
```

---

## 9. What we intentionally do NOT model

| Deferred | Why |
|---|---|
| `user.theme` column | Rejected in D1 — isolation from generated auth tables + room for sibling prefs. |
| Per-workspace themes | Spec §6 out of scope — theme is account-wide (rule 2). |
| Notification preferences | Spec §6 out of scope — every resolved event notifies (minus self-suppress). |
| Profile visibility settings | Spec §6 out of scope — workspace members see member cards, no granularity. |
| Locale / language | Spec §6 out of scope — future `user_preference` column, not a table. |
| Custom avatar presets | Spec §6 out of scope. |
| Server-side image processing (crop/resize/encode) | Rejected in D3 — CSS framing; keeps upload dependency-free. |
| GIF / animated avatars | Rejected in D3 — playback/extraction scope for zero MVP demand. |
| Avatar version history | Overwritten URLs + immutable keys make history pointless — old keys simply dereference. |
| Email/password/session writers | Forbidden by rule 4 (D5 proves it with `.strict()`). |
| Proxy routes to owning domains | Forbidden by rule 5 (§6.5) — links, not endpoints. |
| Private-bucket signed serving | Rejected in D2 — cost on every read path for a non-secret. |

---

## 10. Open product questions — resolved at data layer

| Spec §7 | Decision |
|---|---|
| 1 — avatar replace/delete | **Locked:** replace = new upload + persist + best-effort old cleanup; delete = clear `image` + best-effort cleanup (D4, §6.3). |
| 2 — theme "system" detection | **Locked:** client-side only; server stores the enum (D6, §6.2). |
| 3 — default view | **Locked:** `LIST` both kinds (reaffirmed F4 contract, §6.4). |

---

## 11. References

- Shipyard: `features/settings/spec.md`, `features/auth/data-model.md` (`user.name`/`image`, R2-URL-for-MVP §9), `features/projects/data-model.md` + `api-design.md` §5.2 (`view_preference` ownership, generic endpoints), `features/issues/api-design.md` §5.3 (`ISSUE` scope reuse), `features/members/api-design.md` (delegation targets), `features/workspace/api-design.md` (delegation targets, danger zone), `00-architecture.md` §5/§8, `ADR-001`, `ADR-002`, `ADR-004`, `Implementation Plan.md` F11
- Prisma indexes & referential actions: `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- Cloudflare R2 public buckets: `https://developers.cloudflare.com/r2/buckets/public-buckets/`

---

*Next artifact: `api-design.md` — user-scoped endpoint inventory (`GET/PATCH` profile, `GET/PUT` appearance, `POST/DELETE` avatar; view toggles by reference), session-only guard chain (no workspace context), R2 upload sequence with validation order, error codes, and the delegation link map.*
