# Dashboard — API Design

**Status:** Draft for review
**Last updated:** 2026-09-04
**Sources:** `features/dashboard/spec.md` · `features/dashboard/data-model.md` (locked — `issue_view` only, D1–D7, read contracts §2.2) · `features/workspace/api-design.md` (F2 precedent — `:slug` context, read-when-archived, error envelope) · `features/issues/api-design.md` (F5 precedent — card shapes, detail-read hook site) · `features/projects/api-design.md` (F4 precedent — progress derivation) · `features/cycles/api-design.md` (F7 precedent — active lookup) · `features/comments/api-design.md` (F8 precedent — chronological reads) · `features/auth/api-design.md` (F1 — Better Auth session) · `00-architecture.md` §5–§8 (module dependency rules — cross-module reads via public service APIs) · `ADR-001`–`ADR-003` · `Implementation Plan.md` F9

> **Principle:** identical pipeline, read-only composition:
>
> ```text
> route → validation → permission check → controller → service → owning services → Prisma
> ```
>
> Better Auth handles identity; the F2 workspace context handles authorization (any member may view). The dashboard service owns **no domain rules** — it orchestrates bounded calls into the owning modules' public query services and assembles the composed payload. Per architecture §5, it never queries other modules' tables directly (except its own `issue_view`).

---

## 1. Base path & conventions

| Concern | Choice |
|---|---|
| Base path | `/api/v1/workspaces/:slug/dashboard` — singular resource under the workspace, like `/view-preferences/:scope`. One route, one shape. |
| Next.js proxy | Browser never hits the API directly (ADR-003); `apps/web` forwards `/api/v1/*` → `http://api:4000/api/v1/*`, cookies forwarded. |
| Auth transport | HttpOnly Better Auth session cookie read by `requireSession` (F1) — `req.session.userId` is the only identity input (drives the personal panels). |
| Validation | No body, no query params in MVP — the only input is `:slug` (validated as a slug string at the boundary). Fixed panel bounds live server-side (§5). |
| Envelope | Success: `dashboardSchema` directly. Failure: `{ "error": { "code", "message", "details"? } }` via the global error handler. Empty panels are data (`null`/`[]`), never errors. |
| Workspace context | Reuses F2 `resolveWorkspaceContext(:slug)` verbatim with `rejectArchived: false` — the frozen workspace stays browsable, dashboard included. |
| Trail recording | No endpoint — best-effort side effect inside issues `GET detail` (data-model §6.1/D3). This module owns the recorder function; issues owns the call site. |

---

## 2. Endpoint inventory

One endpoint covers every behavior in `spec.md` §2–§5. Trail recording, drill-down navigation, and badge polling are **not** endpoints here (§5.2). No extras.

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 1 | `GET` | `/api/v1/workspaces/:slug/dashboard` | `requireSession` → member (any role) | §2/§3 Composed payload — My Work (assigned/created/recent), Current Cycle, Active Projects, Recent Activity. Fixed bounds (§5.1). Allowed while workspace `ARCHIVED`. |

> **Why one request, not four:** the Plan's done-criterion #1 — "one dashboard request returns the composed payload." Four round-trips would quadruple proxy hops and leave panels skewing independently anyway; one fan-out inside the API keeps bounds, ordering, and empty-state shapes in a single contract the web renders verbatim.
>
> **Why no query params:** every bound is a locked product decision (10/10/10/1/20), not user preference (spec §6: no customizable layouts). Params would advertise configurability that doesn't exist. Per-panel full browsing lives on the owning pages with their own filters.
>
> **Why no `POST /recently-viewed`:** recording is a server side-effect of viewing (data-model D3) — a second endpoint would let clients diverge and turn a nicety into a failure mode on the detail path.

---

## 3. Context resolution

### 3.1 Workspace route — reuse F2's resolver

Identical to every prior module: `resolveWorkspaceContext(:slug)` resolves the workspace + membership of `req.session.userId` in one query and attaches `req.workspaceContext = { workspaceId, slug, status, role, memberId, userId }`.

- No workspace with slug **or** no membership ⇒ generic `404 WORKSPACE_NOT_FOUND` (no existence leak).
- No role check beyond membership — Members see the same panels as Owners (personal slices differ by `userId`, never by role).
- Workspace `ARCHIVED` ⇒ still `200` — reads are never frozen (§6.1). No `rejectArchived` on this route.
- The personal panels additionally scope by `userId = session.userId` (trail + assigned/created); the shared panels scope by `workspaceId` only.

### 3.2 Owning-service reads (no direct table access)

The dashboard service calls public query functions — never Prisma models it doesn't own:

```text
issuesService.listMyWork(workspaceId, userId)      → assigned + created (bounded, open-only)
dashboardRepository.recentTrail(workspaceId, userId) → own table: top-10 trail ⨝ live issues
cyclesService.getActive(workspaceId)               → 0–1 active cycle + progress
projectsService.listActive(workspaceId)            → active projects + progress each
activityFeed(workspaceId)                          → history leg + comments leg, merged ≤20
```

Each callee enforces its own module's visibility rules (non-archived filters, same-workspace scoping) — the dashboard inherits correctness instead of re-implementing it. `issue_view` is the only table this module's repository touches.

---

## 4. Guard chain (canonical, read-only)

```text
requireSession                     ← F1: valid Better Auth session else 401
  │
resolveWorkspaceContext(slug)      ← F2 shared middleware, rejectArchived: FALSE
  │                                  404 generic on miss/non-membership (no leak)
  │                                  (ARCHIVED workspaces serve 200 — §6.1)
  │
controller → service               ← fan-out (§8.1) with fixed bounds; no role/state preconditions
```

Rules reaffirmed: URL carries workspace context; membership resolved once; no role checks (view surface); no mutations means no confirmation, no archived-freeze, no authorship anywhere in this module.

---

## 5. Request/response contracts

Schemas from `packages/shared` (`data-model.md` §4). The route handler validates `:slug` and nothing else.

### 5.1 The composed payload

| Endpoint | Body / Query | Success response |
|---|---|---|
| #1 dashboard | — (no body, no query) | `200` + `dashboardSchema` `{ workspaceId, myWork: { assigned[≤10], created[≤10], recentlyViewed[≤10 + viewedAt] }, currentCycle: cycleCard \| null, activeProjects[≤20], recentActivity[≤20, newest-first] }` |

Contract details:

- `assigned` / `created`: open issues only (`archivedAt IS NULL AND status != DONE`), `ORDER updatedAt DESC`. Blocked/overdue arrive as row flags for client badges — no separate fields.
- `recentlyViewed`: live issues from the caller's trail, `ORDER viewedAt DESC`, each carrying `viewedAt`; archived issues included with their flag (landing renders read-only); deleted issues absent via cascade.
- `currentCycle`: the active non-archived cycle with inline `progress`, or `null` (designed empty state — never 404).
- `activeProjects`: active non-archived projects with inline `progress` each, F4 list ordering; empty array when none.
- `recentActivity`: `dashboardActivityItemSchema[]`, `(createdAt DESC, id DESC)`, sources restricted to the data-model D4 kinds (issue-history + comment-created — no project/cycle kinds until the Activity Log lands).
- Skew note: panels are read in parallel without a shared snapshot transaction — two panels may straddle a concurrent write by milliseconds. Acceptable in MVP (navigation-time page, no polling); documented, not hidden.

### 5.2 Non-endpoints (explicitly not routes)

| Concern | Served by | Notes |
|---|---|---|
| Trail recording | Issues `GET detail` side effect (F5 #2 + F9 recorder, data-model §6.1) | Best-effort upsert+bump+prune; never fails the read; no HTTP surface. |
| Drill-down | Owning pages (`/issues/:id`, `/cycles/:id`, `/projects/:id`) | Dashboard cards carry ids/slugs only — navigation is client routing. |
| Badge polling | Notifications `GET unread-count` (F6 #2) | Independent loop; dashboard loads on navigation only (spec §3.3). |
| Full browsing | Owning list endpoints (issues/projects/cycles with filters) | Panels are glimpses with "view all" links, not paginated lists. |

---

## 6. Archived / state matrices

### 6.1 Workspace-level (`workspace.status = ARCHIVED`)

| Endpoint | While ARCHIVED | Rationale |
|---|---|---|
| #1 dashboard | ✅ allowed, all four panels | Frozen workspaces stay browsable (F2); panels derive from frozen-but-visible rows |
| Trail recording | ✅ continues | Personal history, not container mutation (data-model §6.5) |

No `409 WORKSPACE_ARCHIVED` exists on this route — it is a `GET` with zero container mutations.

### 6.2 Row-state tolerance (no rejection axis)

The dashboard rejects nothing for row state — it renders through it:

| Row state | Panel behavior |
|---|---|
| Issue archived (trail/activity) | Included with `archivedAt` flag; landing page shows its own banner |
| Issue deleted | Absent (cascade) — trail join drops it, activity source row is gone with it |
| No active cycle | `currentCycle: null` + empty state |
| No projects / no work / empty trail / quiet workspace | `[]` groups + designed empty states (rule 5) |
| Actor deleted (activity) | "Former member" fallback via `actorId SetNull` legacy |

---

## 7. Error codes (Dashboard module)

Global error handler converts typed domain errors; controllers never build envelopes by hand.

| Code | HTTP | When | Notes |
|---|---|---|---|
| `WORKSPACE_NOT_FOUND` | 404 | Unknown `:slug` or caller not a member — deliberately identical | No existence leak (§3.1) |
| `UNAUTHENTICATED` | 401 | Missing/expired session cookie | F1 `requireSession` |
| `RATE_LIMITED` | 429 | Global limiter (wiring finalized at F12) | `Retry-After` header |

Deliberately tiny: no `VALIDATION_ERROR` surface (no body/query to validate beyond the slug shape), no `FORBIDDEN_ROLE` (no roles), no `*_ARCHIVED` 409s (no freeze axis), no `NOT_FOUND` for empty panels (emptiness is data). A panel-source failure (DB error in one leg) is a `500` via the global handler — partial payloads with error placeholders are rejected: a half-composed hub misleads more than an honest error page with retry.

---

## 8. Sequences

### 8.1 Dashboard load (spec §4.1 — the only sequence that matters)

```text
Member → GET /api/v1/workspaces/:slug/dashboard
→ requireSession ✓ → resolveWorkspaceContext ✓ (any member, ARCHIVED ok) → no body to validate
→ service fan-out (parallel, each bounded — no N+1, no shared tx):
     ├─ issuesService.listMyWork(ws, me) → assigned[≤10] + created[≤10]
     ├─ dashboardRepository.recentTrail(ws, me) ⨝ issues → recent[≤10]
     ├─ cyclesService.getActive(ws) → cycle|null (+ progress COUNTs)
     ├─ projectsService.listActive(ws) → projects[] (+ progress COUNTs each)
     └─ activityFeed(ws) → history[≤20] + comments[≤20] → merge → take 20
→ 200 dashboardSchema → web renders four panels or their empty states
→ any leg throws → 500 + error page with retry (no partial payloads, §7)
```

### 8.2 Drill down and return (spec §4.2–§4.3 — recording round-trip)

```text
Member clicks assigned card → GET .../issues/:issueId (F5 #2)
→ 200 card + best-effort: UPSERT issue_view (me, issue) bump + prune-50 (never blocks the 200)
Member navigates back → GET .../dashboard → recent[0] is the just-opened issue
→ revisit bumps it to top again; 51st distinct view prunes the oldest
```

### 8.3 First-run / empty workspace

```text
New member → GET .../dashboard → 200 {
  myWork: { assigned: [], created: [], recentlyViewed: [] },
  currentCycle: null, activeProjects: [], recentActivity: []
} → four designed empty states (each with its CTA: create issue / start cycle / create project)
→ 200, never 404-for-empty (rule 5)
```

### 8.4 Archived workspace browsing

```text
Member → GET .../dashboard on ARCHIVED workspace → 200, all panels from frozen rows
→ cards link to read-only detail pages (each with its own archived banner)
→ trail recording continues (personal, unaffected)
```

---

## 9. Module layout

### 9.1 API — `apps/api/src/features/dashboard/`

```text
features/dashboard/
├── routes.ts        # single path def → middleware chain → controller
├── schemas.ts       # slug param coercion only; payload shapes live in packages/shared
├── controller.ts    # HTTP concerns only: resolve context, call service, map result/errors
├── service.ts       # orchestration only: bounded parallel fan-out, merge/sort/take for activity,
│                    # empty-state assembly; NO domain rules, NO cross-module Prisma
├── repository.ts    # issue_view ONLY: recentTrail read, recordView upsert+prune (called from issues)
└── errors.ts        # ~empty: WORKSPACE_NOT_FOUND passthrough; panel failures are 500s (no partials)
```

Shared guards reused:

```text
common/guards/
├── require-session.ts           # (F1)
└── workspace-context.ts         # (F2) resolveWorkspaceContext(:slug, rejectArchived: false)
```

Cross-module read contracts (owning sides enforce their rules):

```text
issuesService.listMyWork(workspaceId, userId)   → assigned + created (F5 owns the query)
dashboardRepository.recentTrail(workspaceId, userId) → own table ⨝ issues (F9 owns)
cyclesService.getActive(workspaceId)            → active cycle + progress (F7 owns)
projectsService.listActive(workspaceId)         → active projects + progress (F4 owns)
activityFeed(workspaceId)                       → history + comments legs (F5/F8 tables, F9 merges)
dashboardRepository.recordView(userId, issueId, workspaceId) → called by issues GET detail (F5 call site)
```

### 9.2 Shared — `packages/shared/src/dashboard/`

Re-exports from `data-model.md` §4 — the canonical place:

- Groups: `dashboardMyWorkSchema`, `dashboardCycleSchema`, `dashboardProjectsSchema`
- Activity: `dashboardActivityKindSchema`, `dashboardActivityItemSchema`
- Composed: `dashboardSchema`
- Entity cards re-exported from their modules (never redefined): `issueCardSchema`, `cycleCardSchema`, `projectCardSchema`, `issueAssigneeCardSchema`

### 9.3 Web — `apps/web`

| Surface | Route | Reads |
|---|---|---|
| Dashboard hub | `/w/:slug` (workspace landing) | #1 once per navigation (no polling); four panel components from the single payload |
| My Work | hub section | Three groups with badges (blocked/overdue from row flags) + "view all issues" links to filtered issue pages |
| Current Cycle | hub card | Progress bar + dates + link to cycle page; `null` ⇒ empty state with start-cycle CTA (Owner/Admin only) |
| Active Projects | hub section | Progress bars + links; empty ⇒ create-project CTA (Owner/Admin only) |
| Recent Activity | hub feed | Newest-first items with actor + issue links; empty ⇒ quiet-workspace state |

Data access via a single TanStack Query (`useDashboard`, fetch on navigation/focus only — never polled). Mutations: none in this module. All surfaces ship with loading (skeleton panels), error (retryable page, §7 no-partials), empty (per-panel CTAs), and permission-aware states (create/start CTAs hidden for `MEMBER`).

---

## 10. Testing strategy

Three layers (mirrors prior §10s). Tooling provisioned by F1/F2; no new deps.

### 10.1 API integration tests

Supertest against `createApp()`, real Postgres via Testcontainers + migrations. Seeded helpers: all prior creators plus `recordView(user, issue)` (via issues detail reads, exercising the real hook — never direct inserts except for prune-soak setup).

| Case | Covered by |
|---|---|
| Happy path — four panels composed in one `200` | payload-shape assertion against `dashboardSchema` |
| Unauthenticated | `401 UNAUTHENTICATED` |
| Non-member (real slug, foreign user) | `404 WORKSPACE_NOT_FOUND` byte-equal to unknown-slug (leak test) |
| Member sees same panels as Owner (role-irrelevant) | `200` + identical shared panels; personal panels differ by user |
| Empty workspace — all panels empty/null | `200` with empty-state shapes (never 404) |
| My Work bounds — 15 assigned open ⇒ 10 returned; closed/archived excluded | count + exclusion assertions |
| Assigned = me only (teammate's issues absent from my groups) | isolation assertions |
| Trail — detail views populate recent in bump order; revisit re-bumps; 51st prunes oldest | sequence assertions through real `GET issue` calls |
| Trail — deleted issue drops out; archived issue stays flagged | cascade + flag assertions |
| Trail — recording failure (forced hook error) still returns `200` detail | best-effort assertion (rule 4) |
| Current cycle — active renders + progress; none ⇒ `null`; second-active impossible (F7 index) | state assertions |
| Active projects — only `ACTIVE` non-archived, each with progress; planned/completed excluded | filter assertions |
| Activity — issue-history + comment events newest-first ≤20; no project/cycle kinds present | kind-whitelist + ordering + bound assertions |
| Activity — actor-deleted events render fallback (not dropped) | fallback assertions |
| Archived workspace — full `200` + trail still records | no-freeze assertions |
| Cross-workspace — other workspace's rows never appear in any panel | isolation assertions across two workspaces |
| No N+1 — seed 10/10/10 + 20 activity, assert DB round-trip count bounded (query-count spy) | performance-guard assertion |
| Skew tolerance — concurrent write mid-load never 500s | smoke assertion (no snapshot tx required) |

### 10.2 Component tests (web) — MSW mock of `/api/v1/workspaces/:slug/dashboard`

| Surface | Cases |
|---|---|
| Hub | Renders four panels from `dashboardSchema`; skeleton loading; retryable error page (no partial panels); per-panel empty states with CTAs |
| My Work | Three groups render; blocked/overdue badges from row flags; teammate rows never render (contract-level, mocked) |
| Cycle/projects | Progress bars from inline progress; `null` cycle ⇒ empty + CTA visibility per role (start/create hidden for `MEMBER`) |
| Activity | Newest-first items with actor/issue links; comment items link `#comment-<id>`; unknown kinds (future Activity Log) render a generic row, never crash |
| Error envelope rendering | MSW-served `{error:{code,message}}` renders friendly retry, never raw dumps |

Rules: components never re-implement rules (bounds/visibility are API-shaped). Tests assert wire behavior + rendered state.

### 10.3 End-to-end journey — golden path

Playwright against the composed stack (web + api + Postgres, reset between runs).

**Journey — hub lifecycle (core)**

```text
1. Owner + Maya (Member) exist with a project, an active cycle, and issues (F3–F5/F7)
2. Maya signs in → lands on /w/:slug → four panels render (assigned/created work, cycle, projects, activity)
3. Maya opens an assigned issue → detail renders → back to hub → it tops Recently Viewed
4. Maya opens 3 more issues → trail order follows; revisits the first → it re-tops
5. Owner moves Maya's issue to DONE → Maya's hub → gone from Assigned, activity shows the change
6. Owner archives the workspace → hub still 200 with all panels; trail still records
7. New user with empty workspace → hub renders four empty states with CTAs, 200 throughout
```

**Negative E2E checks (cheap):**

- **Deep-link isolation:** second user's `/w/:slug` dashboard URL under a non-member session ⇒ generic not-found.
- **No-partials:** forced panel-source failure (test hook) ⇒ full error page with retry, never half-rendered panels.
- **Stale-activity tolerance:** activity item for a since-deleted issue is gone on refetch (no dead links); actor-deleted items show fallback text.

Scope discipline: journey + negatives are the mandatory F9 E2E suite; exhaustive cases stay in 10.1–10.2. Activity Log feed-swap E2E lands with that feature.

---

## 11. Cross-cutting concerns

| Concern | Approach |
|---|---|
| **Rate limiting** | Global limiter only — no per-route budget (single idempotent `GET`, no mutation to abuse; wiring finalized at F12). Trail writes inherit the issues-detail read path (no separate budget). |
| **Freshness model** | Load-on-navigation + focus refetch; no polling, no realtime (spec §3.3). Skew across panels is accepted and documented (§5.1) — the hub is a glimpse, owning pages are the truth. |
| **Performance budget** | Fixed fan-out (6 bounded queries max) + capped per-project progress counts; batch actor loads; N+1 asserted in tests (§10.1). No snapshot tx (read skew beats lock contention on a landing page). |
| **Pagination** | None — panels are fixed glimpses; drill-down pages own pagination (issues cursor, comments cursor). `recentActivity` bound 20 is a display cap, not a page. |
| **Empty vs error** | Emptiness is always `200` data (§6.2); only auth/scope/infra failures are errors (§7). The web never renders error UI for `null`/`[]`. |
| **Activity evolution** | Panel kinds are a closed set until the Activity Log lands; the web renders unknown kinds generically (§10.2) so the feed-swap needs no flag day. |
| **Trail privacy** | `userId`-scoped reads only; no admin view, no workspace-visible variant (spec Q1). GDPR erase flows through `userId Cascade` (§2.1). |

---

## 12. References

- Shipyard: `features/dashboard/spec.md`, `features/dashboard/data-model.md`, `features/workspace/api-design.md` (context, envelope, read-when-archived), `features/issues/api-design.md` (cards, detail hook site), `features/projects/api-design.md` (progress derivation), `features/cycles/api-design.md` (active lookup), `features/comments/api-design.md` (chronological reads, permalink convention), `features/auth/api-design.md` (session), `00-architecture.md` §5–§8, `ADR-001`–`ADR-003`, `Implementation Plan.md` F9
- Prisma: referential actions, indexes — `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`

---

*Next artifact: implementation (plan §5 Steps 3–7) — Prisma migration (`issue_view` + back-relations, no raw SQL) → module code (routes/controller/service + `issue_view` repository + owning-service read contracts + issues-detail hook) → web slice (hub, four panels, skeletons, empty states) → tests → `pnpm check`. The Activity Log feature, when planned, replaces §6.4's derivation as the activity source without changing the panel contract shape.*
