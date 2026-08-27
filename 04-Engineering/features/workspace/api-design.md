# Workspace — API Design

**Status:** Approved for implementation (F2)
**Last updated:** 2026-08-27
**Sources:** `features/workspace/spec.md` · `features/workspace/data-model.md` (locked) · `features/auth/api-design.md` (F1 precedent) · `00-architecture.md` §5–§8 · `ADR-002` (shared contracts) · `ADR-003` (Next → API proxy) · `Implementation Plan.md` F2

> **Principle:** Unlike F1 (where Better Auth provided every endpoint), **every route in this module is hand-written Shipyard code**, following the canonical pipeline:
>
> ```text
> route → validation → permission check → controller → service → repository → Prisma
> ```
>
> Better Auth handles identity only. This module owns workspace behavior end to end and establishes the guard-chain pattern that F3–F11 copy.

---

## 1. Base path & conventions

| Concern | Choice |
|---|---|
| Base path | `/api/v1/workspaces` (per ADR-001 versioning; Express router mounted by `apps/api` composition root) |
| Item addressing | `:slug` in URLs (`/api/v1/workspaces/:slug`) per data-model D3 — stable, immutable, disambiguates duplicate names; inter-table references stay on `id` |
| Next.js proxy | Browser never hits the API directly (ADR-003); `apps/web` forwards `/api/v1/*` → `http://api:4000/api/v1/*`, cookies forwarded; Caddy exposes only `web:3000` |
| Auth transport | HttpOnly Better Auth session cookie read by session middleware from F1 (`requireSession`); nothing new |
| Validation | Zod schemas from `packages/shared` (`data-model.md` §4) at the route boundary; API rejects anything the client UI would never send |
| Envelope | Success: resource JSON directly. Failure: `{ "error": { "code": "...", "message": "...", "details"? } }` matching the F1 global error handler |

---

## 2. Endpoint inventory

Seven endpoints cover every behavior in spec §2–§5. No extras.

### 2.1 Collection (not workspace-scoped — these operate on *all* of the user's memberships)

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 1 | `POST` | `/api/v1/workspaces` | `requireSession` (verified users only) | §3.1 Create workspace + atomic Owner membership (one transaction). Returns full workspace detail. |
| 2 | `GET` | `/api/v1/workspaces` | `requireSession` | §3.3 List workspaces the user belongs to (name+icon+role for the switcher; drives post-auth routing: 0 → onboarding, 1 → dashboard, N → selection). Order: most recently accessed in client state; server returns `createdAt` desc until F11 adds preference storage. |

### 2.2 Item (workspace-scoped via `:slug`; all go through the §4 guard chain)

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 3 | `GET` | `/api/v1/workspaces/:slug` | `requireSession` → member | §2/§3.3 Workspace detail card. Allowed while archived (owners see the read-only archived view). |
| 4 | `PATCH` | `/api/v1/workspaces/:slug` | `requireSession` → member → Owner* | §2 Rename / change icon. Rejected while archived (read-only). |
| 5 | `POST` | `/api/v1/workspaces/:slug/archive` | `requireSession` → member → Owner | §3.2 Archive, confirmed request body. Sets `ARCHIVED` + `archivedAt`. |
| 6 | `POST` | `/api/v1/workspaces/:slug/restore` | `requireSession` → member → Owner | §3.2 Restore, confirmed request body. Back to `ACTIVE`; history preserved. |
| 7 | `DELETE` | `/api/v1/workspaces/:slug` | `requireSession` → member → Owner | §3.2 Permanent delete: requires `status = ARCHIVED` + exact-name confirmation body. Irreversible single transaction. |


---

## 3. Workspace context resolution (`:slug` → context)

The single lookup used by every item endpoint:

```text
req.params.slug  ──findUnique──▶  workspace (by slug)  +  membership of req.session.userId on that workspace
```

- **One query** joins/lookups both; then the guard chain evaluates:
  - No workspace with that slug **OR** no membership row ⇒ identical generic `404 WORKSPACE_NOT_FOUND` response. This is the no-existence-leak requirement: a non-member and a typo are indistinguishable at the boundary.
  - Membership exists but role insufficient for the route ⇒ `403 FORBIDDEN_ROLE`.
  - Membership exists, role OK, workspace archived, route not in {GET, restore, delete} ⇒ `409 WORKSPACE_ARCHIVED`.
- Resolved context is attached to the request object once (`req.workspaceContext = { workspaceId, status, role }`) so controller/service never re-resolve — one authoritative resolution per request.
- Implemented as an Express middleware in `apps/api/src/common` (shared guard), parameterized by required role — it is the prototype every later feature imports.

---

## 4. Guard chain (canonical, per Implementation Plan §1.4)

```text
requireSession                     ← F1 middleware: valid Better Auth session else 401
  │
  ├─ collection routes (#1–#2): done — operate on session user directly
  │
resolveWorkspaceContext(slug)      ← §3 above; shared middleware, workspace module owns config
  │                                  404 generic on miss/non-membership (no leak)
  ├─ requireRole(Owner)?            ← only #4–#7; composition-level check using resolved context
  │
controller → service               ← service revalidates state preconditions (archived checks,
                                     name-match on delete) even though guards ran first;
                                     defense in depth, same transaction as writes
```

Rules this establishes for all future milestones:

1. The URL carries workspace context; there is no hidden server-side "active workspace" state.
2. Membership resolution happens exactly once per request, generically, with leak-free responses.
3. Role/state checks live in named guards or service preconditions — never inline ad-hoc queries inside controllers.

## 5. Request/response contracts

All schemas from `packages/shared` (`data-model.md` §4). Route handlers validate bodies/params before anything else.

| Endpoint | Body | Success response |
|---|---|---|
| #1 create | `createWorkspaceSchema` `{ name, icon? }` | `201` + `workspaceDetailSchema` (role: `"OWNER"`) |
| #2 list | — | `200` + `{ workspaces: workspaceCardSchema[] }` |
| #3 get | — | `200` + `workspaceDetailSchema` |
| #4 update | `updateWorkspaceSchema` `{ name?, icon? }` (≥1 field) | `200` + `workspaceDetailSchema` |
| #5 archive | `{ confirm: true }` (literal) | `200` + `workspaceDetailSchema` (`ARCHIVED`) |
| #6 restore | `{ confirm: true }` (literal) | `200` + `workspaceDetailSchema` (`ACTIVE`) |
| #7 delete | `deleteWorkspaceSchema` `{ confirmName }` | `204` empty |

Confirmation semantics: archive/restore/delete are destructive-enough that the spec marks them "(Owner, confirmed)". The server rejects a missing literal confirmation flag with `400 CONFIRMATION_REQUIRED` — client dialogs may add friction, but the API enforces its own minimum. For delete, `confirmName` must byte-equal the current stored name after trimming; mismatch ⇒ `400 NAME_MISMATCH` with no hint of correct value beyond what the client already displays.

## 6. Archived read-only enforcement matrix

`status = ARCHIVED` flips the whole workspace read-only except lifecycle exits:

| Endpoint | While ARCHIVED | Rationale (spec §3.2) |
|---|---|---|
| #3 get | ✅ allowed | Owners see the read-only archived view |
| #4 update | ❌ `409 WORKSPACE_ARCHIVED` | No edits to a dormant container |
| #5 archive | ❌ `409 INVALID_STATUS_TRANSITION` | Already archived — idempotent-noise rejected rather than silently repeated |
| #6 restore | ✅ allowed | The exit ramp back to active |
| #7 delete | ✅ allowed | The only other exit — permanent removal |
| Future F3+ endpoints (invites, projects…) | inherit reject-on-archived via shared guard | members can't act inside a frozen container |


---

## 7. Error codes

| Code | HTTP | When | Notes |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | Zod body/param failure (name length, icon key not in `WORKSPACE_ICON_KEYS`, empty patch) | `details` lists field paths |
| `CONFIRMATION_REQUIRED` | 400 | Archive/restore/delete without the literal confirmation flag/body | |
| `NAME_MISMATCH` | 400 | Delete `confirmName` ≠ stored trimmed name | No correction hints |
| `WORKSPACE_NOT_FOUND` | 404 | Unknown slug **or** caller isn't a member — deliberately identical | No existence leak (§3) |
| `FORBIDDEN_ROLE` | 403 | Member but not Owner on #4–#7 | Matters from F3 onward; F2 members are all Owners so effectively unreachable in prod until then — still tested |
| `WORKSPACE_ARCHIVED` | 409 | Mutating op on archived workspace (#4) | Restorable via #6 |
| `INVALID_STATUS_TRANSITION` | 409 | Archive on already-archived (#5), restore on already-active (#6) | Controlled transitions, not generic updates |
| `UNAUTHENTICATED` | 401 | Missing/expired session cookie | Handled by F1 `requireSession`, documented here for completeness |
| `RATE_LIMITED` | 429 | Global API limiter (wiring finalized at F12; auth-level limiting exists already) | |

All errors flow through the global Express error handler with the standard envelope (§1). Services throw typed domain errors; handlers never build envelopes by hand.

## 8. Sequences

### 8.1 Creation → landing (onboarding)

```text
Browser: /onboarding create form → POST /api/v1/workspaces {name, icon}
→ requireSession ✓ → Zod validate → service.create():
    tx { insert workspace (slug gen + collision retry); insert workspace_member OWNER }
→ 201 detail → client routes into /w/:slug (dashboard once F9 exists; placeholder shell before)
```

### 8.2 Post-auth routing decision (from F1, completed here)

```text
After sign-in/OAuth callback → GET /api/v1/workspaces (requireSession)
→ memberships list = ∅        ⇒ redirect /onboarding (create-or-leave loop per resolved spec Q3)
                 = exactly 1  ⇒ redirect /w/:slug (dashboard)
                 = several    ⇒ redirect /workspace-selection (switcher page)
This is Next.js routing logic reading endpoint #2 — the API exposes data, never redirects.
```

### 8.3 Switching

```text
Switcher UI (name+icon+role rows) → pick → client navigates to /w/:slug/...
→ every subsequent item request carries :slug in URL → §3 resolution re-runs per request.
No "active workspace" mutation, no server state to clear — switching is purely client-side navigation.
```

### 8.4 Danger zone: archive → delete

```text
Settings/DangerZone → POST .../archive {confirm:true}   (Owner)     → archived view, read-only everywhere
→ later DELETE /api/v1/workspaces/:slug {confirmName:"exact name"}
→ guards ✓ (Owner, ARCHIVED) → service preconditions ✓
→ single tx: delete memberships → delete workspace → FK cascades handle current/future children
→ 204. Failure anywhere ⇒ full rollback, archived workspace untouched (spec §3.2).
```

---

## 9. Module layout

### 9.1 API — `apps/api/src/features/workspace/`

```text
features/workspace/
├── routes.ts        # router: path defs → middleware chain → controller; Zod validated at entry
├── schemas.ts       # route-local param/query coercion; shared request/response shapes live in packages/shared
├── controller.ts    # HTTP concerns only: parse req, call service, map result/errors to responses
├── service.ts       # business rules, state transitions, transactions (create tx, delete tx)
└── repository.ts    # Prisma access only; no business decisions leak here
```

Shared middleware added by this module:

```text
common/guards/
├── require-session.ts           # (exists from F1)
├── workspace-context.ts         # resolveWorkspaceContext(:slug) — §3; the reusable prototype
└── require-workspace-role.ts    # role check against resolved context
```

> Naming note: the repo uses `src/features/…`; `Implementation Plan.md` §5 Step 4 sketches `modules/<feature>/`. Same concept — when F2 code lands, sync the plan's wording to `features/` rather than renaming code.

### 9.2 Shared — `packages/shared/src/workspace/`

Enums (`workspaceStatusSchema`, `workspaceRoleSchema`), request bodies, `workspaceCardSchema` / `workspaceDetailSchema`, and `WORKSPACE_ICON_KEYS` + `iconSchema` — the canonical Lucide key list and its validator, single-sourced here: the API validates request icons against it and the web builds its `IconPair` (`key → LucideIcon`) render map from it.

### 9.3 Web — `apps/web`

| Surface | Route | Reads/Writes |
|---|---|---|
| Onboarding create | `/onboarding` | #1 create |
| Workspace selection | `/workspace-selection` | #2 list |
| Application shell + switcher | `/w/:slug/*` layout | #2 list, context from URL params |
| Workspace settings (rename/icon) | `/w/:slug/settings` | #3 get, #4 update |
| Danger zone | `/w/:slug/settings/danger` | #5 archive, #6 restore, #7 delete |
| Archived view wrapper | `/w/:slug/*` (status-aware) | read-only affordances per §6 |

Data access via TanStack Query hooks; mutations optimistic only for rename/icon with rollback on failure (plan §1.6 pattern; archive/delete always pessimistic + refetch). All states required per screen: loading, error, empty (0 workspaces), permission-aware.

## 10. Testing strategy

Three layers, each owned where it belongs (maps to plan §5 Step 6). All tooling below was already provisioned by F1's `apps/web` package.json — no new dependencies.

### 10.1 API integration tests

Supertest against `app.ts`, real Postgres via Testcontainers, migrations applied by `vitest.global-setup.ts`.

| Case | Covered by |
|---|---|
| Happy paths ×7 endpoints | Supertest suite against `app.ts` (F1 harness, Testcontainers Postgres) |
| Invalid input (name length, bad icon key, empty patch, missing confirm) | per-endpoint 400 assertions |
| Unauthenticated access ×7 | 401 `UNAUTHENTICATED` |
| Non-member item access (real slug, foreign user) | 404 identical to unknown-slug — **assert the two responses are byte-equal** (leak test) |
| Wrong role on owner-only routes | seeded non-owner membership row + 403 (testable now even though prod can't reach it until F3) |
| Archived write attempts (#4/#5-on-archived) | 409 codes |
| Cross-actor concurrency: two creates race slug | retry logic test |
| Delete atomicity: force child-table failure in test hook | assert full rollback, archived workspace intact |
| Restore preserves history | archivedAt retained, memberships/timestamps unchanged |
| Cascade contract | create descendant rows (only member rows exist in F2) → delete → children gone, user rows alive |

### 10.2 Component tests (web) — UI behavior with mocked API

**Setup:** Vitest + jsdom + Testing Library; an MSW server intercepts `/api/v1/workspaces*` with per-test handlers whose payloads are built from the real `packages/shared` schemas — components are tested against contract-shaped data, not hand-rolled fixtures that can drift from the API.

| Surface | Cases |
|---|---|
| Onboarding create form | Submit issues `POST #1` with a schema-valid body (assert via MSW request spy); invalid name shows inline error **without firing a request**; success navigates into `/w/:slug`; mocked `400 VALIDATION_ERROR` maps code → field message |
| Workspace selection / switcher | Rows render name + icon + role from mocked list; icon key resolves through `IconPair` to the right Lucide component; empty list renders the 0-workspaces state |
| Post-auth routing decision | Pure routing logic tested directly: mock list ∅ / exactly 1 / several ⇒ onboarding / workspace / selection targets asserted |
| Settings rename & icon | Form prefilled from detail payload; rename applied optimistically and **rolled back** on mocked `500`; archived status disables edit affordances |
| Danger zone | Archive dialog sends `{confirm:true}` once confirmed; delete requires typed exact name — mismatch blocks submit client-side *and* mocked `400 NAME_MISMATCH` renders error state; `409 WORKSPACE_ARCHIVED` / `INVALID_STATUS_TRANSITION` show recoverable messaging (restore path visible) |
| Error envelope rendering | Every surface renders MSW-served `{error:{code,message}}` envelopes as friendly states, never raw dumps |

Rules: components must never re-implement business rules (plan §1.2) — component tests assert wire behavior and rendered state, not internals. Coverage target: every screen ships with loading, error, empty, and permission/archived state tests before merge.

### 10.3 End-to-end journey — one golden path kept green at all times

Playwright (`pnpm --filter @shipyard/web test:e2e`) against the locally composed stack (web + api + Postgres, DB reset/re-migrated between runs). In local dev the verification email is captured by the test email adapter (F1's mode) instead of Resend.

```text
1. Sign up with unique email/password
2. Verify via the dev-captured email link
3. Land in onboarding          ← asserts post-auth rule: 0 workspaces
4. Create workspace (name + pick an icon key)
5. Land in /w/:slug            ← shell shows picked icon + owner context
6. Rename + change icon → hard refresh → changes persist
7. Archive → read-only view; restore → active again
8. Sign out → sign in → routed straight into the single workspace   ← 1-membership dashboard rule
```

Plus two negative E2E checks (cheap, high-value):

- **Deep-link isolation:** second browser session (another test user) opens the first user's `/w/:slug/...` URL ⇒ generic not-found, identical for unknown slugs.
- **Delete double-guard:** attempting delete without archive fails; after archiving, delete requires typing the exact name — wrong text is rejected, workspace survives.

Scope discipline: the golden journey above is the **only** mandatory E2E suite for F2; exhaustive cases stay in layers 10.1–10.2 where they run in milliseconds rather than minutes. CI wiring for the Playwright job is finalized during F12 per plan §4.

---

## 11. References

- Shipyard: `features/workspace/spec.md`, `features/workspace/data-model.md`, `features/auth/api-design.md` (envelope/error precedent), `00-architecture.md` §5–§8, `ADR-002`, `ADR-003`, `Implementation Plan.md` F2
- Express middleware patterns & error handling conventions follow the existing `apps/api/src/common` utilities built in F1

---

*Next artifact: implementation itself (plan §5 Steps 3–7) — Prisma migration → module code → web slice → tests → `pnpm check`. No further design doc needed; data-model + api-design cover F2's technical design.*
