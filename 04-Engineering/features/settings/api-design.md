# Settings — API Design

**Status:** Draft for review
**Last updated:** 2026-09-04
**Sources:** `features/settings/spec.md` · `features/settings/data-model.md` (locked — `user_preference`, R2 conventions, D1–D7) · `features/auth/api-design.md` (F1 — session, Auth-exclusive identity flows) · `features/projects/api-design.md` §5.2 (F4 view-preference endpoints — reused) · `features/workspace/api-design.md` (F2 — delegation targets, danger zone) · `features/members/api-design.md` (F3 — delegation targets) · `00-architecture.md` §5–§8 · `ADR-001`–`ADR-004` · `Implementation Plan.md` F11
**Owner:** `apps/api` — hand-written Shipyard code through the canonical pipeline (`route → validation → session check → controller → service → repository → Prisma/R2`).

> **Two scope families, stated once:** account routes (`/api/v1/settings/*` — session-only, no workspace, work pre-workspace in onboarding) and reused workspace routes (F4 view-preference endpoints — unchanged). Delegated sections are client links, never proxy endpoints (rule 5).

---

## 1. Base path & conventions

| Concern | Choice |
|---|---|
| Base paths | `/api/v1/settings/profile`, `/api/v1/settings/appearance`, `/api/v1/settings/avatar` — user-scoped, no `:slug`. View toggles stay on `/api/v1/workspaces/:slug/view-preferences/:scope` (F4, untouched). |
| Next.js proxy | Browser never hits the API directly (ADR-003); `apps/web` forwards `/api/v1/*` → `http://api:4000/api/v1/*`, cookies forwarded. |
| Auth transport | HttpOnly Better Auth session cookie read by `requireSession` (F1) — `req.session.userId` scopes every read and write to self. No workspace context, no roles — your settings are yours regardless of membership. |
| Validation | Zod from `packages/shared` at the route boundary; profile schema is `.strict()` (an `email` key is a `400`, D5). Avatar validated pre-buffer against MIME allowlist + 2MB cap (D3). |
| Envelope | Success: resource JSON directly. Failure: `{ "error": { "code", "message", "details"? } }` via the global error handler. |
| Avatar transport | `multipart/form-data`, single `avatar` file field. Request-body cap sits below the file cap at middleware so floods die before buffering. |
| R2 | Server-side SDK with boot-validated env; tests inject the in-memory fake (§9.1). Public-bucket reads need no API involvement — only upload/delete touch this module. |

---

## 2. Endpoint inventory

Six endpoints: five owned here covering spec §2–§3.3, plus the reused F4 pair documented for the shell map (§5.2). Delegated sections are links (§5.3). No extras.

### 2.1 Account — profile, appearance, avatar (session-only)

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 1 | `GET` | `/api/v1/settings/profile` | `requireSession` (self) | §2/§4.1 Profile card — name, read-only email, avatar URL, verification flag. |
| 2 | `PATCH` | `/api/v1/settings/profile` | `requireSession` (self) | §3.1/§4.1 Rename — `updateProfileSchema` (`.strict()` — email rejected, D5). Applies account-wide instantly. |
| 3 | `GET` | `/api/v1/settings/appearance` | `requireSession` (self) | §3.2/§4.2 Theme — `{ theme }`, `SYSTEM` when no row (D6). |
| 4 | `PUT` | `/api/v1/settings/appearance` | `requireSession` (self) | §3.2/§4.2 Set theme — upsert (lazy row creation). Client applies instantly. |
| 5 | `POST` | `/api/v1/settings/avatar` | `requireSession` (self) | §3.1/§4.1 Upload — validate → R2 put → persist URL → best-effort old cleanup (D4). Returns `{ image }`. |
| 6 | `DELETE` | `/api/v1/settings/avatar` | `requireSession` (self) | §3.1 Clear — `{ confirm: true }` → null `image` + best-effort object delete. Returns updated profile card. |

> **Why session-only with no workspace:** onboarding users (zero workspaces) must set name/avatar/theme before ever joining one — a `:slug` scope would lock them out of their own account page. Self-scoping (`WHERE id = session.userId`) replaces workspace isolation here; there is no other user's row to leak to.
>
> **Why `DELETE` avatar takes `confirm: true`:** uniform delete discipline across the codebase (labels/cycles/comments/notifications all confirm) — clearing is one click to regret and one literal to prevent.

---

## 3. Context resolution

One lookup, owned by F1's session middleware — no F2 workspace resolution runs on #1–#6:

```text
req.session.userId ──self scope──▶ User WHERE id = session.userId (+ UserPreference upsert/read)
```

- No session ⇒ `401 UNAUTHENTICATED` before anything else.
- The user row always exists under a valid session (F1 invariant) — #1/#3 never 404 (appearance-absent reads as `SYSTEM`, D6).
- R2 object keys embed `userId` (`avatars/:userId/:uuid`) so a key can never address another account's prefix — defense in depth beneath the self-scope.

---

## 4. Guard chain (canonical — session variant)

```text
requireSession                     ← F1: valid Better Auth session else 401
  │
selfScope                          ← every query pins id/userId = session.userId
  │                                  (profile/avatar/appearance — no workspace, no role)
controller → service               ← .strict() profile shape, theme enum, avatar validation order (§8.1)
```

What this chain omits (vs F2–F10): no workspace context, no roles, no archived checks (account writes are never container mutations), no authorship (self *is* the authorizer). The `.strict()` profile schema is the email-domain firewall (D5).

---

## 5. Request/response contracts

### 5.1 Owned routes

| Endpoint | Body | Success response |
|---|---|---|
| #1 get profile | — | `200` + `profileCardSchema` |
| #2 rename | `updateProfileSchema` `{ name }` (`.strict()` — anything else 400s) | `200` + `profileCardSchema` |
| #3 get theme | — | `200` + `appearanceSchema` (`SYSTEM` when no row) |
| #4 set theme | `setAppearanceSchema` `{ theme }` | `200` + `appearanceSchema` (upserted) |
| #5 upload avatar | `multipart [avatar]` — MIME ∈ allowlist, ext matches, bytes ≤2MB | `201` + `avatarCardSchema` `{ image }` (`201` — a storage object is created) |
| #6 clear avatar | `{ confirm: true }` | `200` + `profileCardSchema` (`image: null`) |

Validation & behavior details:

- `#2`: `name` trimmed, 1–100 (D7); `email` (or any unknown key) ⇒ `400 VALIDATION_ERROR` via `.strict()` — the test-observable proof of rule 4. Same-name set is a no-op `200` (no write, F5 no-op discipline).
- `#4`: unknown enum ⇒ `400`; upsert creates the `user_preference` row on first set (no backfill anywhere).
- `#5` validation order (each gate fails before the next costs anything): presence of the `avatar` part → MIME ∈ allowlist → extension matches MIME → bytes ≤2MB (pre-buffer cap) → R2 put → persist URL → best-effort old-key delete. Any gate ⇒ `400 VALIDATION_ERROR` with the field/reason in `details`; R2 outages ⇒ `500` (storage is the operation, not hygiene — contrast D4 cleanups, which never surface).
- `#5` returns `201` (a new immutable object exists at a new key); `#6` returns the profile card (so the shell updates name/email/verification display in one round-trip).
- `#6` missing literal ⇒ `400 CONFIRMATION_REQUIRED`; already-null image ⇒ `200` unchanged card (idempotent clear, not an error).

### 5.2 Reused routes — view toggles (F4, unchanged)

`GET/PUT /api/v1/workspaces/:slug/view-preferences/:scope` (`PROJECT`/`ISSUE`) serve the List⇄Kanban toggles on owning pages exactly as specified in projects api-design §5.2. F11 defines no new scope, shape, or default — this table row exists so reviewers confirm nothing is missing: nothing is.

### 5.3 Delegated sections — link map (not endpoints)

| Settings tab | Route (client navigation) | Owning API |
|---|---|---|
| Security — password/email | Auth screens (F1) | Better Auth flows; Settings sends zero requests |
| Workspace details | `/w/:slug/settings` | Workspace #3/#4 (Owner-only updates) |
| Members & invitations | `/w/:slug/members` | Members #1–#10 |
| Danger zone | `/w/:slug/settings/danger` | Workspace #5–#7 |
| Leave workspace | User menu / settings | Members #5 |

Any `POST /settings/change-password`-shaped proxy would duplicate guards it cannot own (rule 5) — route-table tests assert the only settings paths are #1–#6 (§10.1).

---

## 6. Archived / state matrices

| Situation | #1–#6 |
|---|---|
| Zero workspaces (onboarding) | ✅ all allowed — the point of session-only scope |
| Any workspace `ARCHIVED` | ✅ unaffected — account writes are never container mutations (§6.6 of data-model) |
| Email unverified | ✅ allowed for profile/appearance/avatar (F1 gates Auth-gated actions, not self-presentation); Auth flows own verification UX |

No 409 axis exists in this module.

---

## 7. Error codes (Settings module)

| Code | HTTP | When | Notes |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | Zod failure (bad/empty/overlong name, unknown profile key incl. `email`, bad theme enum, avatar missing part / bad MIME / ext mismatch / over-size) | `details` lists field paths; the `email`-key case is the rule-4 proof |
| `CONFIRMATION_REQUIRED` | 400 | #6 without literal `confirm: true` | Uniform delete discipline |
| `UNAUTHENTICATED` | 401 | Missing/expired session cookie | F1 |
| `RATE_LIMITED` | 429 | Avatar upload bucket (strict: 10/hr per user — uploads cost storage; wiring F12, global limiter exists) | `Retry-After` |

No `NOT_FOUND` (self rows always resolve; appearance-absent reads default), no `FORBIDDEN_ROLE` (no roles), no `WORKSPACE_*` (no workspace scope). R2 outages surface as `500` via the global handler + Sentry (unexpected-infrastructure class).

---

## 8. Sequences

### 8.1 Avatar upload — validation order is the design

```text
Member → POST .../avatar  multipart[avatar: photo.png, 1.2MB]
→ requireSession ✓ (self) → gates in order:
     part present? ─MIME ∈ {jpeg,png,webp}? ─ext matches MIME? ─bytes ≤ 2MB?
→ key = avatars/:userId/:uuid.png → R2 PutObject (public, immutable headers)
→ UPDATE "user" SET image = url → 201 { image: url }
→ best-effort R2 delete OLD key (fail → log + Sentry breadcrumb, 201 stands — D4)
→ every card in the product shows the new avatar (all join user.image live)
```

### 8.2 Rename + theme (account-wide propagation)

```text
Member → PATCH .../profile {name:"Maya Chen"} → UPDATE user.name → 200 card
→ comments/issues/members/activity rows render the new name (all live joins; activity summaries stay frozen by design)

Member → PUT .../appearance {theme:"DARK"} → UPSERT user_preference → 200 { theme:"DARK" }
→ client applies instantly + persists; reload reads GET #3 → DARK (no flash of SYSTEM)
```

### 8.3 Strictness proof (rule 4, test-observable)

```text
Member → PATCH .../profile {name:"Maya", email:"new@x.com"}
→ .strict() rejects unknown key → 400 VALIDATION_ERROR (details: ["email"])
→ user.email untouched (only Auth change-email flow writes it, after re-auth + verification)
```

### 8.4 Clear avatar (idempotent)

```text
Member → DELETE .../avatar {confirm:true}
→ UPDATE user SET image=NULL → best-effort R2 delete → 200 card (image: null)
→ repeat → 200 unchanged (idempotent, not 404/409)
→ OAuth users clearing a provider avatar simply null it — next OAuth sync may refill per provider behavior (documented, not fought)
```

---

## 9. Module layout

### 9.1 API — `apps/api/src/features/settings/`

```text
features/settings/
├── routes.ts        # account path defs → self-scope → controller; Zod validated at entry
│                    # (#1–#6; NO workspace routes, NO proxy routes — §5.2/§5.3)
├── schemas.ts       # multipart avatar coercion (MIME/ext/size gates); shared shapes in packages/shared
├── controller.ts    # HTTP concerns only: parse session/body/file, call service, map result/errors
├── service.ts       # profile/appearance writers, avatar validate→put→persist→cleanup flow
├── repository.ts    # user.name/image writes + user_preference read/upsert (Auth tables touched narrowly)
├── r2.ts            # R2 adapter interface (put/delete) — real SDK in prod, in-memory fake in tests/dev-optional
└── errors.ts        # typed domain errors → global handler maps to §7
```

Shared guards reused: `require-session.ts` (F1) only — workspace-context/role guards are deliberately absent (reviewable absence, §4).

Cross-module posture (all negative — the design): no imports from Auth internals (narrow Prisma writes only), no calls into Workspace/Members/Projects/Issues services, no new view-preference code.

### 9.2 Shared — `packages/shared/src/settings/`

`themePreferenceSchema`, `displayNameSchema`, `avatarMimeAllowlist`, `AVATAR_MAX_BYTES`, `updateProfileSchema` (`.strict()`), `setAppearanceSchema`, `profileCardSchema`, `appearanceSchema`, `avatarCardSchema`. View-preference contracts stay in their F4/F5 homes (re-exported for shell convenience only).

### 9.3 Web — `apps/web`

| Surface | Route | Reads/Writes |
|---|---|---|
| Account settings | `/settings/account` (or shell section) | #1 get, #2 rename, #5 upload (preview + progress), #6 clear |
| Appearance | `/settings/appearance` | #3 get, #4 set (instant apply + persist; `SYSTEM` tracks OS client-side) |
| Delegated tabs | Security / Workspace / Members / Danger zone | Links per §5.3 — zero settings-API traffic |
| View toggles | Owning issues/projects pages | F4 endpoints (existing hooks); shell never reimplements |
| App shell | Global | Theme consumed at root (persisted choice, no flash); avatar from profile query |

Data access via TanStack Query (profile/appearance queries, pessimistic mutations — account truth must be authoritative; avatar upload with progress states). All surfaces ship with loading, error, empty (no avatar → initials fallback), and permission-aware **omission** (rule 6 — tabs/sections absent by role, e.g. danger zone hidden for non-Owners, never disabled buttons).

---

## 10. Testing strategy

### 10.1 API integration tests

Supertest + Testcontainers + in-memory fake R2 (map of key→bytes; failure-injection hook for outage/cleanup-failure paths).

| Case | Covered by |
|---|---|
| Happy paths ×6 endpoints | per-route suites |
| Invalid input (empty/overlong name, bad theme, unknown profile key incl. `email`, missing avatar part, bad MIME, ext mismatch, over-size) | `400 VALIDATION_ERROR` (+ details assertions; `email`-key case is the rule-4 proof) |
| Missing `confirm: true` (#6) | `400 CONFIRMATION_REQUIRED` |
| Unauthenticated ×6 | `401` |
| Rename → visible account-wide (comment author + member card in two workspaces) | propagation assertions |
| Same-name rename → no-op `200` | no-write assertions |
| Theme — absent ⇒ `SYSTEM`; set ⇒ upsert round-trip; invalid ⇒ `400`; second set overwrites (one row) | default + upsert assertions |
| Avatar — happy `201` + `user.image` persisted + fake-R2 object present with immutable headers | storage + DB assertions |
| Avatar — replace persists new URL + old key deleted (best-effort path) | rotation assertions |
| Avatar — cleanup-failure injection still `201` with new URL (D4) | hygiene-never-surfaces assertions |
| Avatar — R2-outage injection ⇒ `500`, `user.image` unchanged (no persist without put) | ordering assertions |
| Avatar — clear nulls + deletes object; repeat clear ⇒ `200` unchanged | idempotency assertions |
| Avatar — oversized (2MB+1) and wrong-type (gif/exe) rejected pre-R2 (fake adapter untouched) | gate-order assertions |
| Settings paths are exactly #1–#6 (proxy-shaped paths 404) | route-table assertions (rule-5 proof) |
| Rate-limit bucket on #5 (config-level test, wiring F12) | limit assertions |
| Cross-user isolation — one user's avatar upload never touches another's `image` (self-scope) | isolation assertions (no workspace needed) |

### 10.2 Component tests — MSW mocks of `/api/v1/settings/*` + F4 view-preference routes

Account form (prefilled; `email` field rendered read-only, never submitted); rename `400`-on-`email` mapping; appearance segmented control (instant preview + persist; `SYSTEM` follows OS mock); avatar picker (type/size client gates mirror server; progress; success swaps image; R2-`500` shows retry); clear-avatar confirm flow; view toggles call F4 endpoints (existing coverage referenced); delegated tabs render as links (no settings-API traffic asserted via MSW spy); omission states per role (danger-zone tab absent for Member — never disabled); initials fallback with no avatar; error-envelope rendering throughout.

### 10.3 End-to-end journey

```text
1. New user signs up → onboarding → sets display name + avatar pre-workspace (session-only proof)
2. Creates workspace → dashboard shell shows name + avatar everywhere (propagation proof)
3. Switches theme DARK → reload → still DARK (persistence proof); SYSTEM tracks OS mock
4. Issues page → Kanban → reload → Kanban persists (F4 toggle untouched by F11)
5. Renames → member directory + old comments show new name; activity summaries keep frozen names (contrast proof)
6. Replaces avatar → new image everywhere; clears → initials fallback
```

Negatives: crafted `email`-in-profile PATCH → 400, email unchanged; oversized upload → 400 pre-R2; danger-zone tab absent (not disabled) for Member; archived workspace changes nothing about account screens.

---

## 11. Cross-cutting concerns

| Concern | Approach |
|---|---|
| **Rate limiting** | Upload bucket strict (10/hr per user — storage costs; wiring F12). Profile/theme inherit the global limiter (cheap single-row writes). |
| **R2 posture** | Public bucket, unguessable keys, immutable cache headers (D2). Private-bucket migration path preserved: keys are already random per object, so a future flip only changes serving, never naming. |
| **Orphan hygiene** | D4 crash windows leave inert unreferenced objects; post-MVP janitor lists prefix vs referenced URLs (no table tracks them — §7). |
| **Theme flashing** | Root consumes the persisted choice before first paint (server-injected or blocking query); `SYSTEM` resolves client-side. No flash-of-default is a release-gate UI check. |
| **Omission discipline** | Rule 6 is web-enforced from membership role; API needs no counterpart (self-routes have no role axis). Tests assert absence, not disabled-ness. |
| **No new preferences without a row-field** | Future prefs extend `user_preference` columns (defaulted, nullable) — the table is the throttle against pref sprawl. |

---

## 12. References

- Shipyard: `features/settings/spec.md`, `features/settings/data-model.md`, `features/auth/data-model.md` + `api-design.md` (identity ownership, session), `features/projects/api-design.md` §5.2 (view-preference endpoints), `features/issues/api-design.md` §5.3 (scope reuse), `features/workspace/api-design.md` + `features/members/api-design.md` (delegation targets), `00-architecture.md` §5/§8, `ADR-001`, `ADR-002`, `ADR-004`, `Implementation Plan.md` F11
- Cloudflare R2 public buckets: `https://developers.cloudflare.com/r2/buckets/public-buckets/`
- Prisma: `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`

---

*Next artifact: implementation (plan §5 Steps 3–7) — Prisma migration (`user_preference` + enum, no auth-table edits) → module code (routes/controller/service/repository + R2 adapter + fake for tests + shared schemas) → web slice (account/appearance/delegated shell, toggles untouched, omission states) → tests → `pnpm check`. F12 hardening (rate-limit wiring, logging, health, deploy) follows.*
