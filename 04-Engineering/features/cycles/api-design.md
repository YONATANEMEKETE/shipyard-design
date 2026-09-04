# Cycles — API Design

**Status:** Draft for review
**Last updated:** 2026-09-04
**Sources:** `features/cycles/spec.md` · `features/cycles/data-model.md` (locked — `cycle` + `issue.cycleId SetNull` + `CYCLE_CHANGED`, D1–D8) · `features/workspace/api-design.md` (F2 precedent — `:slug` context, guard chain, archived matrix, error envelope) · `features/members/api-design.md` (F3 precedent — RBAC pipeline, `confirm: true` convention) · `features/projects/api-design.md` (F4 precedent — archive dimension, name-conflict pattern, `SetNull` unassign leg) · `features/issues/api-design.md` (F5 precedent — issue-level writes, history pattern, `?archived` + filter conventions) · `features/auth/api-design.md` (F1 — Better Auth session) · `00-architecture.md` §5–§8 · `ADR-001`–`ADR-003` · `Implementation Plan.md` F7

> **Principle:** identical to projects (F4) — every route is hand-written Shipyard code through the canonical pipeline:
>
> ```text
> route → validation → permission check → controller → service → repository → Prisma
> ```
>
> Better Auth handles identity; this module owns **authorization** for cycle data — who may read cycles (any member) and who may create/edit/lifecycle them (Owner/Admin only). No new auth primitive; it reuses the F2/F3 guard chain verbatim.
>
> **Locked product decisions for F7 (2026-09-04):** delete confirms with `{ confirm: true }` (not `confirmName`); no dedicated `/active` endpoint — Dashboard reads `GET /cycles?status=ACTIVE`; date edits on `ACTIVE` cycles allowed but re-validate overlap/uniqueness; attaching issues to `COMPLETED` cycles allowed (correction path); list defaults to `sort=startDate order=asc`.

---

## 1. Base path & conventions

| Concern | Choice |
|---|---|
| Base path | `/api/v1/workspaces/:slug/cycles` and `/api/v1/workspaces/:slug/cycles/:cycleId` — mirrors projects; `:slug` is the F2 immutable workspace token; `:cycleId` is the cycle's `cuid()` (data-model D2 — no cycle slug, same reasoning as projects D5). Issue↔cycle writes live on the issues resource (`PATCH /issues/:issueId` extended with `cycleId`, §5.2) — there is no cycle-side `/cycles/:id/issues` writer. |
| Next.js proxy | Browser never hits the API directly (ADR-003); `apps/web` forwards `/api/v1/*` → `http://api:4000/api/v1/*`, cookies forwarded. |
| Auth transport | HttpOnly Better Auth session cookie read by `requireSession` (F1) — `req.session.userId` is the only identity input. |
| Validation | Zod schemas from `packages/shared` (`data-model.md` §4) at the route boundary. Dates are `YYYY-MM-DD` strings end-to-end (`@db.Date`, D4). |
| Envelope | Success: resource JSON directly (or `{ cycles: [...] }` for collections). Failure: `{ "error": { "code", "message", "details"? } }` via the global error handler. Guard-failure conflicts carry `details.conflictingCycle: cycleCard` so the UI can name it (data-model §10 — spec Q3 resolved). |
| Workspace context | Reuses F2 `resolveWorkspaceContext(:slug)` verbatim — one authoritative resolution per request, leak-free `404 WORKSPACE_NOT_FOUND`. |
| Archived enforcement (workspace) | Mutating routes use `resolveWorkspaceContext({ rejectArchived: true })`; `GET` routes pass `rejectArchived: false`. |
| Cycle-level read-only (archived cycle / completed cycle) | Enforced in the **service** against `cycle.archivedAt` + `cycle.status` (§6.2) — restore remains allowed on archived cycles; reopen/archive remain allowed on completed cycles per the lifecycle matrix. |

---

## 2. Endpoint inventory

Ten endpoints cover every behavior in `spec.md` §2–§5 and `data-model.md` §6. Issue assignment, progress computation, dashboard aggregation, and full-text search are **not** endpoints here (§5.2, §11). No extras.

### 2.1 Workspace-scoped — cycle CRUD & lifecycle

All under `/api/v1/workspaces/:slug/cycles...`, all through the §4 guard chain.

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 1 | `GET` | `/api/v1/workspaces/:slug/cycles` | `requireSession` → member (any role) | §2/§4 List cycles. Query: `status` filter (active lookup is `?status=ACTIVE`, no dedicated endpoint per locked decision); `?archived=true` returns archived only. Sort `sort`/`order` (default `startDate asc`, chronological). No pagination — cycles are few (like projects/labels). Cards carry inline `progress` (derived, D8). |
| 2 | `GET` | `/api/v1/workspaces/:slug/cycles/:cycleId` | `requireSession` → member (any) | §2/§4 Detail — card + `goal` + `progress`. Returns archived cycles too (historical reference). Scoped: `:cycleId` validated against `:slug`. No embedded issue list — clients fetch `GET /issues?cycleId=` (§5.2). |
| 3 | `POST` | `/api/v1/workspaces/:slug/cycles` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | §4.1 Create — lands `PLANNED`. Body `createCycleSchema`. Pre-checks name uniqueness (D3) + overlap (D5) for friendly errors; DB constraints are the backstop. |
| 4 | `PATCH` | `/api/v1/workspaces/:slug/cycles/:cycleId` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | §3.1 Edit `name`/`goal`/`startDate`/`endDate`. No `status` field (D2 — transitions are named actions #5–#7). Allowed on `PLANNED` and `ACTIVE` (dates re-validate D3/D5); rejected on `COMPLETED`/archived (§6.2). |
| 5 | `POST` | `/api/v1/workspaces/:slug/cycles/:cycleId/start` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | `PLANNED → ACTIVE`. Guards: no other `ACTIVE` (D6) + range still non-overlapping (D5). Body `{ confirm: true }`. |
| 6 | `POST` | `/api/v1/workspaces/:slug/cycles/:cycleId/complete` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | `ACTIVE → COMPLETED`. No issue writes (rule 9). Body `{ confirm: true }`. |
| 7 | `POST` | `/api/v1/workspaces/:slug/cycles/:cycleId/reopen` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | `COMPLETED → ACTIVE`. Same guards as Start, re-evaluated at reopen time. Body `{ confirm: true }`. |
| 8 | `POST` | `/api/v1/workspaces/:slug/cycles/:cycleId/archive` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | Archive — only `PLANNED`/`COMPLETED` (`ACTIVE` ⇒ `409 COMPLETE_FIRST`). Sets `archivedAt`, keeps `status`. Body `{ confirm: true }`. |
| 9 | `POST` | `/api/v1/workspaces/:slug/cycles/:cycleId/restore` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | Restore — clears `archivedAt`, returns to stored `status`. Exclusion constraint re-evaluated; conflict ⇒ `409 CYCLE_OVERLAP`, stays archived. Body `{ confirm: true }`. |
| 10 | `DELETE` | `/api/v1/workspaces/:slug/cycles/:cycleId` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | Permanent delete — only future `PLANNED` (`status=PLANNED AND startDate>today`), per locked decision confirms with `{ confirm: true }` (not `confirmName`). One tx: unassign issues (`SetNull` + `CYCLE_CHANGED` rows) + physical row delete. Name released. |

> **Why no generic status write:** spec rule 5 forbids `PLANNED → COMPLETED` skips and `ACTIVE → PLANNED` regressions. Named actions (#5–#7) make illegal transitions unrepresentable at the route layer and let error codes name the failed guard (D2).
>
> **Why no `/active` endpoint:** at most one `ACTIVE` non-archived row exists per workspace (D6). `GET #1?status=ACTIVE` returns zero-or-one cards — the Dashboard (F9) reads that. A dedicated endpoint would duplicate filter logic for zero benefit (locked decision).
>
> **Why no cycle-side issue writer:** spec §3.3 fixes add/remove as issue-level actions. The write path is issues `PATCH { cycleId }` (§5.2); the cycles module owns only the validation contract the issues service calls — same cross-module-service pattern as F3→F4 `transferOwnedProjects`.

---

## 3. Context resolution

### 3.1 Workspace-scoped routes (#1–#10) — reuse F2's resolver

Identical to projects/issues: `resolveWorkspaceContext(:slug)` resolves the workspace + membership of `req.session.userId` in one query and attaches `req.workspaceContext = { workspaceId, slug, status, role, memberId, userId }`.

- No workspace with slug **or** no membership ⇒ generic `404 WORKSPACE_NOT_FOUND` (no existence leak).
- Membership exists, role insufficient (Member on a write route) ⇒ `403 FORBIDDEN_ROLE`.
- Workspace `ARCHIVED` + `rejectArchived: true` ⇒ `409 WORKSPACE_ARCHIVED`.
- The module-owned `:cycleId` lookup then scopes to `workspaceId` (never cross-workspace) ⇒ `404 CYCLE_NOT_FOUND`.

### 3.2 Cycle detail resolution (#2, #4–#10)

```text
req.params.cycleId ──findFirst(where: { id, workspaceId: context.workspaceId })──▶ cycle
```

- No row in this workspace ⇒ `404 CYCLE_NOT_FOUND` (generic; a cycle id from another workspace is indistinguishable).
- The cycle row carries `status` + `archivedAt` — service uses both for the lifecycle matrix (§6.2). Never resolved against a different `workspaceId`.
- Guard-failure responses for `ANOTHER_ACTIVE_EXISTS` / `CYCLE_OVERLAP` include the conflicting row as `details.conflictingCycle` (a `cycleCard`), fetched in the same tx as the pre-check so the UI can render "complete *Sprint 12* first".

### 3.3 Issues-leg resolution (§5.2)

Owned by the issues module; the cycles service exposes a read-only validator `assertCycleAssignable(workspaceId, cycleId, tx)` used in-tx: cycle exists in the same workspace else `404 CYCLE_NOT_IN_WORKSPACE`, `archivedAt` set else `409 CYCLE_ARCHIVED`. `COMPLETED` passes (locked decision — correction path, D7).

---

## 4. Guard chain (canonical, mirrors projects §4)

### 4.1 Cycle routes (#1–#10)

```text
requireSession                     ← F1: valid Better Auth session else 401
  │
resolveWorkspaceContext(slug)      ← F2 shared middleware
  │                                  404 generic on miss/non-membership (no leak)
  │                                  409 WORKSPACE_ARCHIVED when rejectArchived && ARCHIVED
  ├─ requireWorkspaceRole("OWNER","ADMIN")   ← write routes #3–#10
  │                                            (view routes #1–#2 accept any member)
  │
cycleById :cycleId                 ← module lookup scoped to workspaceId → 404 CYCLE_NOT_FOUND
  │
service preconditions              ← lifecycle matrix (§6.2): status + archivedAt + confirm:true
                                       + name uniqueness + overlap/active pre-checks
                                       — inside the same transaction as writes;
                                       DB constraints D3/D5/D6 are the race backstop
```

Rules reaffirmed (inherited from F2–F5): URL carries workspace context, no hidden server state; membership resolved once; role/state checks in named guards or service preconditions, never inline ad-hoc controller queries; workspace-archived workspaces read-only at the guard layer; archived/completed-cycle read-only reasserted in the service.

---

## 5. Request/response contracts

Schemas from `packages/shared` (`data-model.md` §4). Route handlers validate bodies, params, and query before anything else.

### 5.1 Cycles

| Endpoint | Body / Query | Success response |
|---|---|---|
| #1 list | query `?status=PLANNED\|ACTIVE\|COMPLETED` (single, optional; omitted ⇒ all statuses; `ACTIVE` is the Dashboard current-cycle lookup) · `?archived=true` (default false; true ⇒ archived only) · `?sort=createdAt\|name\|startDate\|endDate\|status` (default `startDate`) · `?order=asc\|desc` (default `asc`, chronological — deliberate divergence from projects `desc`) | `200` + `{ cycles: cycleCardSchema[] }` (each with inline `progress`) sorted per sort/order |
| #2 detail | — | `200` + `cycleDetailSchema` (card + `goal`) |
| #3 create | `createCycleSchema` `{ name (required), goal?, startDate: YYYY-MM-DD (required), endDate: YYYY-MM-DD (required, >= startDate) }` | `201` + `cycleDetailSchema` (`status: PLANNED`, empty `progress`) |
| #4 update | `updateCycleSchema` `{ name?, goal? (nullable — null clears), startDate?, endDate? }` (≥1 field; no `status` field by design) | `200` + `cycleDetailSchema` (updated) |
| #5 start | `{ confirm: true }` | `200` + `cycleDetailSchema` (`status: ACTIVE`) |
| #6 complete | `{ confirm: true }` | `200` + `cycleDetailSchema` (`status: COMPLETED`; issues untouched) |
| #7 reopen | `{ confirm: true }` | `200` + `cycleDetailSchema` (`status: ACTIVE`) |
| #8 archive | `{ confirm: true }` | `200` + `cycleDetailSchema` (`archivedAt` set, `status` unchanged) |
| #9 restore | `{ confirm: true }` | `200` + `cycleDetailSchema` (`archivedAt` null, prior `status` intact) |
| #10 delete | `{ confirm: true }` | `200` + `{ deletedCycleId, unassignedIssues: number }` — issues survive, unassigned in the same tx |

Validation & precondition details:

- `#3`/`#4`: `name` trimmed by Zod then uniqueness re-checked against the D3 `lower(name)` functional index — duplicate (including archived rows, which reserve names) ⇒ `409 CYCLE_NAME_CONFLICT` with `details.conflictingCycle`. `endDate < startDate` ⇒ `400 VALIDATION_ERROR`. Same-day `start == end` valid (one-day iteration, inclusive semantics D4/D5).
- `#3`/`#4`/`#9`: overlap pre-checked in service (`daterange(start,end,'[]') &&` against non-archived siblings) ⇒ `409 CYCLE_OVERLAP` with `details.conflictingCycle`; the D5 exclusion constraint backstops races (concurrent conflicting commits — one wins, the loser maps to the same code).
- `#4`: allowed on `PLANNED` and `ACTIVE` non-archived (locked decision — `ACTIVE` date edits re-validate D5; e.g. extending the active range into a sibling fails `CYCLE_OVERLAP`). Rejected on `COMPLETED` ⇒ `409 CYCLE_READ_ONLY` and on archived ⇒ `409 CYCLE_ARCHIVED` (§6.2). Partial body: omitted fields leave as is; `goal: null` explicitly clears.
- `#5`/`#7`: preconditions `status` correct (`PLANNED` for start, `COMPLETED` for reopen) + `archivedAt IS NULL` + no other `ACTIVE` non-archived sibling (D6 pre-check ⇒ `409 ANOTHER_ACTIVE_EXISTS` + conflicting card) + range still non-overlapping (D5 pre-check ⇒ `409 CYCLE_OVERLAP`). Wrong-status calls ⇒ `409 INVALID_CYCLE_TRANSITION` (e.g. start on `ACTIVE`, reopen on `PLANNED`).
- `#6`: preconditions `status = ACTIVE` + non-archived; else `409 INVALID_CYCLE_TRANSITION`. No issue writes in the tx (rule 9).
- `#8`/`#9`/`#10`/`#5`/`#6`/`#7` require literal `confirm: true` (missing ⇒ `400 CONFIRMATION_REQUIRED`, same precedent as workspace/members/projects). `#10` uses `confirm: true` per locked decision — divergence from projects `confirmName` / issues `confirmIdentifier` is intentional: delete is gated to future-`PLANNED` (narrow blast radius, issues only unassigned) so typed-name friction buys nothing.
- `#8` rejects `ACTIVE` ⇒ `409 COMPLETE_FIRST` (rule 6) and already-archived ⇒ `409 ALREADY_ARCHIVED`. `#9` rejects non-archived ⇒ `409 NOT_ARCHIVED`; overlap on restore ⇒ `409 CYCLE_OVERLAP`, cycle stays archived (rule 8).
- `#10` preconditions `status = PLANNED AND archivedAt IS NULL AND startDate > today(server date, day-precision)` — anything else (Active, Completed, archived, already-started Planned) ⇒ `409 CYCLE_NOT_DELETABLE` with a reason message (single code per data-model §6.6). Runs one `$transaction`: `UPDATE issue SET cycleId=NULL WHERE cycleId=?` (+ `CYCLE_CHANGED` history per affected issue, actor = deleter) and `DELETE FROM cycle`. Name released (D3 over non-deleted rows only).
- `#1`: unknown `status`/`sort`/`order` ⇒ `400 VALIDATION_ERROR`. `?archived=true` lists archived only (boards/lists default to non-archived). No cursor/limit params — unknown pagination params ⇒ `400 VALIDATION_ERROR` (there is no second pagination mode; a server `LIMIT` cap applies for safety, §11).

### 5.2 Issues-leg (extended by F7, owned by the issues module — no new endpoints)

| Endpoint | Body / Query delta | Success response |
|---|---|---|
| `PATCH /api/v1/workspaces/:slug/issues/:issueId` (F5 #4) | `updateIssueSchema += { cycleId: cuid.nullable.optional }` — `null` ⇒ detach; omitted ⇒ leave as is; same-cycle set ⇒ no-op (no write, no history, mirrors F5 assignee/project discipline) | `200` + `issueDetailSchema` (now carrying `cycleId`) |
| `GET /api/v1/workspaces/:slug/issues` (F5 #1) | query += `?cycleId=<cuid>` — ANDs with all existing filters/sort/search/cursor; `?cycleId=` + `?archived=true` reads archived issues of the cycle | unchanged `200` + `{ issues, nextCursor }` |

Preconditions (issues service, calling the cycles validation contract in-tx): attach asserts same-workspace cycle (`404 CYCLE_NOT_IN_WORKSPACE`) and `cycle.archivedAt IS NULL` (`409 CYCLE_ARCHIVED`); issue archived ⇒ `409 ISSUE_ARCHIVED` (F5 matrix); detach always allowed including from `COMPLETED`/archived-association cycles; attach to `COMPLETED` allowed per locked decision (D7 correction path); attach/detach emits `CYCLE_CHANGED { old, new }` history; never changes issue `status`/`blocked`, and completing a cycle never changes its issues (rules 9/11 both directions).

---

## 6. Read-only / archived enforcement matrices

### 6.1 Workspace-level (`workspace.status = ARCHIVED`)

| Endpoint | While ARCHIVED | Rationale |
|---|---|---|
| #1 list, #2 detail | ✅ allowed | Read-only — the frozen workspace stays browsable |
| Issues-leg reads (`GET issues`, history) | ✅ allowed | Same |
| #3–#10 all writes + issues-leg `PATCH { cycleId }` | ❌ `409 WORKSPACE_ARCHIVED` | No cycle edits in a frozen container |

Enforced at the guard layer (`rejectArchived: true`) for #3–#10.

### 6.2 Cycle-level (`cycle.archivedAt` set / `cycle.status` — own lifecycle, active workspace)

Archived **cycles** are read-only historical reference (spec §3.3); `COMPLETED` cycles are row read-only until reopened (rule 7) but still accept issue-level attach/detach (locked decision). Enforced in the service:

| Endpoint | While cycle archived | While `COMPLETED` (non-archived) | Notes |
|---|---|---|---|
| #1 list, #2 detail | ✅ allowed (`#1` only via `?archived=true`) | ✅ allowed | Archived never shown on boards/lists by default |
| #4 update | ❌ `409 CYCLE_ARCHIVED` | ❌ `409 CYCLE_READ_ONLY` | `ACTIVE`/`PLANNED` non-archived only |
| #5 start | ❌ `409 CYCLE_ARCHIVED` (checked before transition) | ❌ `409 INVALID_CYCLE_TRANSITION` | Only `PLANNED` non-archived |
| #6 complete | ❌ `409 CYCLE_ARCHIVED` | ❌ `409 INVALID_CYCLE_TRANSITION` | Only `ACTIVE` non-archived |
| #7 reopen | ❌ `409 CYCLE_ARCHIVED` (restore first) | ✅ (allowed — this is the way out) | Only `COMPLETED` non-archived |
| #8 archive | ❌ `409 ALREADY_ARCHIVED` | ✅ allowed | `PLANNED`/`COMPLETED` non-archived only |
| #9 restore | ✅ (allowed — this is the way out) | ✅ when archived (orthogonal axis) | Requires `archivedAt` set; else `409 NOT_ARCHIVED` |
| #10 delete | ❌ `409 CYCLE_NOT_DELETABLE` | ❌ `409 CYCLE_NOT_DELETABLE` | Only future-`PLANNED` non-archived; archived-`PLANNED` also rejected even if dates still future (restore first, then delete if still eligible) |
| Issues-leg attach to this cycle | ❌ `409 CYCLE_ARCHIVED` | ✅ allowed (locked correction path) | Detach always allowed everywhere |

Defense in depth: service reasserts `archivedAt` + `status` even though the guard already ran.

---

## 7. Error codes (Cycles module)

Global error handler converts typed domain errors; controllers never build envelopes by hand.

| Code | HTTP | When | Notes |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | Zod body/param/query failure (bad name/goal/date, `endDate < startDate`, bad `status`/`sort`/`order`, pagination params, missing body) | `details` lists field paths |
| `CONFIRMATION_REQUIRED` | 400 | #5–#10 without literal `confirm: true` | Same precedent as workspace/members/projects |
| `WORKSPACE_NOT_FOUND` | 404 | Unknown `:slug` or caller not a member — deliberately identical | No existence leak (§3.1) |
| `CYCLE_NOT_FOUND` | 404 | `:cycleId` not in this workspace | Scoped — not a cross-workspace leak |
| `CYCLE_NOT_IN_WORKSPACE` | 404 | Issues-leg `cycleId` not in this workspace | Scoped (issues module surface, cycles contract defines it) |
| `FORBIDDEN_ROLE` | 403 | Member on a write route (#3–#10) | Tested with seeded roles |
| `CYCLE_NAME_CONFLICT` | 409 | #3/#4 name normalized (trim, case-insensitive) collides, incl. archived rows | D3 functional index + friendly pre-check; `details.conflictingCycle` |
| `CYCLE_OVERLAP` | 409 | #3/#4/#7/#9 range overlaps a non-archived sibling (inclusive bounds) | D5 exclusion + pre-check; `details.conflictingCycle` |
| `ANOTHER_ACTIVE_EXISTS` | 409 | #5/#7 while another `ACTIVE` non-archived cycle holds the slot | D6 partial index + pre-check; `details.conflictingCycle` |
| `INVALID_CYCLE_TRANSITION` | 409 | Lifecycle action on the wrong `status` (e.g. start on `ACTIVE`, complete on `PLANNED`, reopen on `PLANNED`) | Controlled transitions only (rule 5) |
| `COMPLETE_FIRST` | 409 | #8 archive on an `ACTIVE` cycle | Rule 6 — complete before archiving |
| `CYCLE_READ_ONLY` | 409 | #4 update on a `COMPLETED` cycle | Rule 7 — reopen first |
| `CYCLE_NOT_DELETABLE` | 409 | #10 unless future-`PLANNED` non-archived | Single code (rule: Active/Completed/archived/started all land here; message names the reason) |
| `CYCLE_ARCHIVED` | 409 | #4–#8/#10 on an archived cycle; issues-leg attach to an archived cycle | Read-only historical reference |
| `ALREADY_ARCHIVED` | 409 | #8 on an already-archived cycle | |
| `NOT_ARCHIVED` | 409 | #9 on a non-archived cycle | |
| `WORKSPACE_ARCHIVED` | 409 | Mutating op while the workspace is `ARCHIVED` (§6.1) | Restorable via workspace restore |
| `UNAUTHENTICATED` | 401 | Missing/expired session cookie | F1 `requireSession` |
| `RATE_LIMITED` | 429 | Per-route create/lifecycle limits (wiring finalized at F12; global limiter exists) | `Retry-After` header |

---

## 8. Sequences

### 8.1 Create cycle (spec §4.1)

```text
Owner/Admin → POST /api/v1/workspaces/:slug/cycles {name:"Sprint 13", goal:"Checkout", startDate:"2026-09-08", endDate:"2026-09-19"}
→ requireSession ✓ → resolveWorkspaceContext ✓ (OWNER|ADMIN) → Zod validate (endDate >= startDate)
→ service tx {
     normalize name (trim) → assert no CYCLE_NAME_CONFLICT (lower(name), incl. archived)
     assert no CYCLE_OVERLAP vs non-archived siblings (inclusive daterange)
     insert cycle { workspaceId, name, goal, startDate, endDate, status: PLANNED }
   } → 201 detail (empty progress)
→ concurrent conflicting create → loser hits D5 at commit → 409 CYCLE_OVERLAP + conflictingCycle
→ client navigates to /w/:slug/cycles/:id
```

### 8.2 Start / complete / reopen (spec §3.2)

```text
Owner/Admin → POST .../cycles/:id/start {confirm:true}   // PLANNED, non-archived
→ guard ✓ → service tx {
     re-read cycle FOR UPDATE; assert status=PLANNED else 409 INVALID_CYCLE_TRANSITION
     assert no other ACTIVE non-archived sibling else 409 ANOTHER_ACTIVE_EXISTS + card
     assert range still non-overlapping else 409 CYCLE_OVERLAP + card
     UPDATE status=ACTIVE
   } → 200 detail (issues untouched)

Owner/Admin → POST .../cycles/:id/complete {confirm:true} // ACTIVE
→ service: assert ACTIVE → UPDATE status=COMPLETED (no issue writes) → 200 (now read-only per §6.2)

Owner/Admin → POST .../cycles/:id/reopen {confirm:true}  // COMPLETED
→ same guards as Start re-evaluated (later cycle may hold the range/slot)
→ UPDATE status=ACTIVE → 200
→ failure → 4xx + conflictingCycle card → UI names it ("complete Sprint 14 first")
```

### 8.3 Edit dates on ACTIVE (locked decision)

```text
Owner/Admin → PATCH .../cycles/:id {endDate:"2026-09-26"}  // ACTIVE, non-archived
→ guard ✓ → service tx {
     assert status ∈ {PLANNED, ACTIVE} (COMPLETED ⇒ 409 CYCLE_READ_ONLY)
     assert name uniqueness if renamed (incl. archived) else 409 CYCLE_NAME_CONFLICT
     assert new range overlaps nothing non-archived (self excluded) else 409 CYCLE_OVERLAP
     UPDATE row
   } → 200 detail
```

### 8.4 Archive → restore (spec §3.2)

```text
Owner/Admin → POST .../cycles/:id/archive {confirm:true}   // PLANNED or COMPLETED
→ guard ✓ → service: assert archivedAt IS NULL; assert status ≠ ACTIVE else 409 COMPLETE_FIRST
→ UPDATE archivedAt=now() (status untouched) → 200
→ archived row leaves scheduling (D5 WHERE clause) + default lists (?archived=true only)

Owner/Admin → POST .../cycles/:id/restore {confirm:true}
→ service: assert archivedAt IS NOT NULL else 409 NOT_ARCHIVED
→ UPDATE archivedAt=NULL → exclusion re-evaluated at commit
→ conflict with a newer non-archived row ⇒ 409 CYCLE_OVERLAP, stays archived → 200 otherwise (prior status intact)
```

### 8.5 Delete future Planned (spec rule 10)

```text
Owner/Admin → DELETE .../cycles/:id  body {confirm:true}   // PLANNED, non-archived, startDate > today
→ guard ✓ → service tx {
     re-read cycle FOR UPDATE; assert delete-eligibility else 409 CYCLE_NOT_DELETABLE
     UPDATE issue SET cycleId=NULL WHERE cycleId=:id   // F7 leg, live
       + INSERT CYCLE_CHANGED {old: cycleId, new: null} per affected issue (actor = deleter)
     DELETE FROM cycle WHERE id=:id
   } → 200 { deletedCycleId, unassignedIssues: count }
→ any failure → full rollback; cycle and issues unchanged (all-or-nothing)
→ name released for reuse (functional index over non-deleted rows)
```

### 8.6 Issue ↔ cycle assignment (F7 extension, issues-owned)

```text
Member → PATCH .../issues/:issueId {cycleId:"<cuid>"}   // any member (issues RBAC, not cycles RBAC)
→ issues guard (any member, rejectArchived) → issues service tx {
     call cycles assertCycleAssignable(workspaceId, cycleId): same-workspace else 404 CYCLE_NOT_IN_WORKSPACE;
       archived cycle else 409 CYCLE_ARCHIVED (COMPLETED passes — locked correction path)
     assert issue.archivedAt IS NULL else 409 ISSUE_ARCHIVED
     same-cycle set ⇒ no-op (no write, no history)
     else SET cycleId + INSERT CYCLE_CHANGED {old, new}
   } → 200 issueDetail (cycleId set; status/blocked untouched)

Member → PATCH .../issues/:issueId {cycleId:null} → detach always allowed → 200 + CYCLE_CHANGED
Member → GET .../issues?cycleId=<cuid> → cycle issue list (filters/sort/cursor compose)
```

### 8.7 Dashboard current-cycle read (F9 consumer)

```text
GET /api/v1/workspaces/:slug/cycles?status=ACTIVE   // any member
→ zero-or-one card + progress (D6 guarantees ≤1) → Dashboard renders current cycle + counts
→ empty ⇒ no active cycle state
```

---

## 9. Module layout

### 9.1 API — `apps/api/src/features/cycles/`

```text
features/cycles/
├── routes.ts        # router: path defs → middleware chain → controller; Zod validated at entry
│                    # (#1–#10; no issue-writer routes — those stay in features/issues)
├── schemas.ts       # route-local param/query coercion (slug/cycleId params,
│                    # status/archived/sort/order query, confirm:true bodies); shared schemas in packages/shared
├── controller.ts    # HTTP concerns only: parse req/query, call service, map result/errors
├── service.ts       # business rules: name/overlap/active pre-checks, lifecycle matrix,
│                    # archive/restore/delete txs, assertCycleAssignable contract, progress derivation; transactions
├── repository.ts    # Prisma access only (cycle reads/writes, sibling-range lookup, progress counts)
└── errors.ts        # typed domain errors → global handler maps to §7
```

Shared guards reused (not owned by this module):

```text
common/guards/
├── require-session.ts           # (F1)
├── workspace-context.ts         # (F2) resolveWorkspaceContext(:slug)
└── require-workspace-role.ts    # (F2) — used for #3–#10 as requireWorkspaceRole("OWNER","ADMIN")
```

Cross-module contracts:

```text
cyclesService.assertCycleAssignable(workspaceId, cycleId, tx)  → called by issuesService on PATCH { cycleId }
issuesService.unassignOnCycleDelete(cycleId, actorId, tx)       → called by cyclesService in #10 tx
dashboardService (F9) → GET #1?status=ACTIVE + progress (read-only consumer)
```

### 9.2 Shared — `packages/shared/src/cycles/`

Re-exports from `data-model.md` §4 — the canonical place:

- Enums/bounds: `cycleStatusSchema`, `cycleNameSchema`, `cycleGoalSchema`, `cycleDateSchema`
- Request: `createCycleSchema`, `updateCycleSchema`, `cycleLifecycleSchema` (`{ confirm: true }`), list-query schemas (status/archived/sort/order)
- Response: `cycleProgressSchema`, `cycleCardSchema`, `cycleDetailSchema`, delete-result schema (`{ deletedCycleId, unassignedIssues }`)
- Issues-leg delta: `updateIssueSchema += { cycleId }`, `issueCard/Detail += { cycleId }`, `issueHistoryEventSchema += CYCLE_CHANGED`, list-query += `cycleId`

### 9.3 Web — `apps/web`

| Surface | Route | Reads/Writes |
|---|---|---|
| Cycles page (list + toolbar) | `/w/:slug/cycles` (+ `?status=&archived=&sort=&order=`) | #1 list (filters/sort), #3 create (page/global menu) |
| Cycle detail | `/w/:slug/cycles/:cycleId` | #2 detail, #4 update, #5 start, #6 complete, #7 reopen, #8 archive, #9 restore, #10 delete; issue list via `GET issues?cycleId=` |
| Modals | over page / detail | Create, Edit (dates re-validated), Start/Complete/Reopen confirm, Archive/Restore confirm (`confirm:true`), Delete confirm (`confirm:true` + shows `unassignedIssues`) |
| Dashboard widget (F9) | `/w/:slug` (dashboard) | `GET #1?status=ACTIVE` + progress |
| Global create menu | App shell | Create Cycle (Owner/Admin only, permission-filtered) |

Data access via TanStack Query hooks (like members/projects): standard queries for #1/#2 (no infinite-query — unpaginated), mutations pessimistic for create/update/lifecycle/delete (authoritative — no optimistic status flips). Lifecycle buttons render only valid next actions for the current `status`; blocked actions explain the guard (conflicting cycle named from `details.conflictingCycle`).

All surfaces ship with loading, error, empty (no cycles; no cycles matching filter; empty cycle with `progress.percent: null`), and permission-aware states (create/edit/lifecycle/delete affordances hidden for `MEMBER`; archived/completed editors disabled with "restore/reopen to edit" messaging).

---

## 10. Testing strategy

Three layers (mirrors projects/issues §10). Tooling provisioned by F1/F2; no new deps.

### 10.1 API integration tests

Supertest against `createApp()`, real Postgres via Testcontainers + migrations (incl. `btree_gist` + D3/D5/D6 raw SQL). Seeded helpers: `createVerifiedUser`, `createWorkspaceAs(owner)`, `addMember(workspace, user, role)`, `createCycle(workspace, overrides)`, `createIssue(workspace, overrides)`.

| Case | Covered by |
|---|---|
| Happy paths ×10 endpoints | Supertest suite per group (cycles CRUD + lifecycle) |
| Invalid input (empty/overlong name/goal, bad date shape, `endDate < startDate`, bad `status`/`sort`/`order`, pagination params, missing body) | `400 VALIDATION_ERROR` |
| Missing `confirm: true` (#5–#10) | `400 CONFIRMATION_REQUIRED` |
| Unauthenticated ×10 | `401 UNAUTHENTICATED` |
| Non-member access (real slug, foreign user) | `404 WORKSPACE_NOT_FOUND` — assert byte-equal to unknown-slug (leak test) |
| Member on write route (#3–#10) | `403 FORBIDDEN_ROLE` (Member reads 200) |
| Create → lands `PLANNED`, empty progress | `201` + DB assertions |
| Rename collision (case-insensitive, incl. archived rows reserve) | `409 CYCLE_NAME_CONFLICT` + `details.conflictingCycle` |
| Duplicate create after physical delete | succeeds (name freed) |
| Overlap — inclusive bounds (`end Jan 10` + `start Jan 10` conflicts; `start Jan 11` OK) | `409 CYCLE_OVERLAP` + conflicting card |
| Overlap — archived siblings neither block nor are blocked | `201` over archived range succeeds |
| Overlap — concurrent conflicting creates | one `201`, one `409` (exclusion backstop) |
| Overlap — `ACTIVE` date edit into sibling range | `409 CYCLE_OVERLAP` |
| Overlap — restore into occupied range | `409 CYCLE_OVERLAP`, stays archived |
| Single-active — start while another `ACTIVE` | `409 ANOTHER_ACTIVE_EXISTS` + conflicting card |
| Single-active — concurrent starts | one `200`, one `409` (partial-index backstop) |
| Single-active — reopen while another `ACTIVE` | `409 ANOTHER_ACTIVE_EXISTS` |
| Wrong-status transitions (start on `ACTIVE`, complete on `PLANNED`/`COMPLETED`, reopen on `PLANNED`/`ACTIVE`) | `409 INVALID_CYCLE_TRANSITION` |
| Update on `COMPLETED` | `409 CYCLE_READ_ONLY` |
| Update/archived-action on archived cycle | `409 CYCLE_ARCHIVED` / `409 ALREADY_ARCHIVED` / `409 NOT_ARCHIVED` |
| Archive `ACTIVE` | `409 COMPLETE_FIRST` |
| Complete → issues untouched (open issues stay open, same `cycleId`) | `200` + issue-row assertions |
| Archive → restore round trip preserves `status` | `200`, `archivedAt` set/cleared, `status` unchanged |
| Delete — future `PLANNED` unassigns issues (+ `CYCLE_CHANGED` rows), issues alive, count asserted | `200` + `unassignedIssues` + join assertions |
| Delete — Active/Completed/archived/started-`PLANNED` | `409 CYCLE_NOT_DELETABLE` |
| Delete rollback on unassign failure | cycle still present |
| Progress — `total/completed/percent`, `null` when empty, blocked excluded structurally, archived issues excluded | `200` card/detail assertions |
| Issues-leg — attach same-workspace active/`PLANNED`/`COMPLETED` ⇒ `200`; cross-workspace ⇒ `404 CYCLE_NOT_IN_WORKSPACE`; archived cycle ⇒ `409 CYCLE_ARCHIVED`; archived issue ⇒ `409 ISSUE_ARCHIVED`; same-cycle ⇒ no-op (history count unchanged); detach (`null`) always allowed incl. `COMPLETED` | cross-module tests via issues routes |
| Archived workspace writes (#3–#10) | `409 WORKSPACE_ARCHIVED` |
| Archived workspace reads (#1, #2) | `200` |
| Cross-workspace — cycle id from another workspace | `404 CYCLE_NOT_FOUND` scoped |
| List — `status` filter (incl. `ACTIVE` zero-or-one), `archived` flag, `sort`/`order` (default `startDate asc`) | returned sets + ordering assertions |

### 10.2 Component tests (web) — MSW mocks of `/api/v1/workspaces/:slug/cycles*` + `.../issues?cycleId=*`

| Surface | Cases |
|---|---|
| Cycles page | Renders `cycleCardSchema` list with progress; empty states (no cycles / no filter matches); archived view via `?archived=true` |
| Cycle card | Shows name, date range, status badge, progress bar/counts, active highlight |
| Permission-aware toolbar | Create hidden for `MEMBER`; edit/lifecycle/delete controls hidden for `MEMBER` |
| Create modal | Sends `POST #3` (MSW spy asserts schema-valid body incl. `YYYY-MM-DD`); `CYCLE_NAME_CONFLICT`/`CYCLE_OVERLAP` show inline with conflicting cycle named; success navigates to detail |
| Edit modal | Sends `PATCH #4`; rename/overlap conflicts inline; `CYCLE_READ_ONLY`/`CYCLE_ARCHIVED` show restore/reopen path |
| Lifecycle buttons | Only valid next actions rendered per `status`; each sends its `POST` with `confirm:true`; `ANOTHER_ACTIVE_EXISTS`/`CYCLE_OVERLAP`/`COMPLETE_FIRST`/`INVALID_CYCLE_TRANSITION` render the guard + conflicting cycle name; failure leaves row unchanged |
| Archive/restore/delete dialogs | Send `#8`/`#9`/`#10` with `confirm:true`; delete shows `unassignedIssues`; archived card leaves default list; restore from archived view returns it |
| Cycle detail issue list | Fetches `GET issues?cycleId=`; empty-cycle state (`percent: null`); blocked badge never affects progress display |
| Error envelope rendering | Every surface renders MSW-served `{error:{code,message,details}}` as friendly states, never raw dumps |
| Archived workspace/cycle wrappers | All mutating affordances disabled with frozen/archived messaging |

Rules: components never re-implement business rules (e.g., "Member cannot start" is API-enforced; web just hides controls). Tests assert wire behavior + rendered state.

### 10.3 End-to-end journey — golden path

Playwright against the composed stack (web + api + Postgres, reset between runs).

**Journey — cycle lifecycle (core)**

```text
1. Owner + a Member exist in a workspace (F3)
2. Owner creates "Sprint 13" (dates + goal) → lands on detail as PLANNED
3. Member creates two issues and assigns both to the cycle via issue editor → progress 0/2
4. Owner starts the cycle → ACTIVE; tries to start a second PLANNED cycle → blocked with active named
5. Owner edits ACTIVE end date (extend, non-conflicting) → succeeds
6. Member moves one issue to DONE → progress 1/2 (blocked flag would not affect it)
7. Owner completes the cycle → COMPLETED; issues stay as-is (open stays open)
8. Owner attaches a third issue to the COMPLETED cycle (correction path) → allowed
9. Owner reopens → ACTIVE again; then completes → archives → shown under Archived view
10. Owner restores → back with prior status; Member sees everything but no lifecycle controls
11. Owner creates a future PLANNED cycle with an issue, then deletes it (confirm) → cycle gone, issue survives unassigned
```

**Negative E2E checks (cheap):**

- **Overlap:** create "S2" overlapping "S1" (same day bound) → 409 shown inline with conflicting name.
- **Member write attempt:** Member sends `POST #3`/`POST start` directly → 403.
- **Active archive:** attempt archive on `ACTIVE` → 409 `COMPLETE_FIRST`; complete first, then archive succeeds.
- **Cross-workspace leak:** second workspace's cycle id under first workspace's slug → 404.
- **Archived workspace freeze:** archive the workspace, then attempt create/update/lifecycle → 409; pages still readable.

Scope discipline: journey + negatives are the mandatory F7 E2E suite; exhaustive cases stay in 10.1–10.2. Dashboard-current-cycle and search-grouped E2E land with F9/F10.

---

## 11. Cross-cutting concerns

| Concern | Approach |
|---|---|
| **Rate limiting** | Per-route create (10/min per workspace), lifecycle actions (20/min), delete (5/min). Memory for MVP; global limiter exists; wiring finalized at F12. |
| **Validation encoding** | Dates are `YYYY-MM-DD` strings end-to-end (data-model D4, `@db.Date`); api coerces to `Date`, web formats from the same schema. `endDate >= startDate` at Zod; inclusivity enforced by `daterange(...,'[]')`. |
| **Sorting / ordering** | Server applies `sort`/`order` (default `startDate asc` — chronological, locked decision); no client-side re-sort contract. Server never stores manual order. |
| **Pagination** | None — cycles are few (spec: no guardrails on count, but workspace holds tens, not thousands). A `LIMIT` cap at the API layer for safety, not structural pagination. Cycle *issues* paginate via the issues cursor endpoint. |
| **Audit / history** | No cycle-level history table in F7. The audit trail for assignment is per-issue `CYCLE_CHANGED` rows (F5 pattern); `cycle.createdAt`/`updatedAt` cover record-level display. |
| **Progress derivation** | Live `COUNT(*)` pair over `issue WHERE workspaceId AND cycleId AND archivedAt IS NULL` grouped by `status = DONE` (D8). Returned inline on cards/details; never persisted. Blocked never a predicate. |
| **Search** | Cycle list has no `q` param in F7; global search lands with F10 (generated `tsvector` on `cycle(name, goal)` + GIN, grouped endpoint) without changing these contracts. |
| **Notifications** | None for cycles in MVP (notifications spec §3.1: assignment + mentions only; explicitly no cycle-change notifications). No hook emitted from #3–#10. |

---

## 12. References

- Shipyard: `features/cycles/spec.md`, `features/cycles/data-model.md`, `features/workspace/api-design.md` (guard chain, archived matrix, envelope), `features/members/api-design.md` (RBAC pipeline, `confirm: true`), `features/projects/api-design.md` (archive dimension, name-conflict pattern, `SetNull` unassign leg), `features/issues/api-design.md` (issue-level writes, history pattern, filter conventions), `features/auth/api-design.md` (session), `00-architecture.md` §5–§8, `ADR-001`–`ADR-003`, `Implementation Plan.md` F7
- Prisma: referential actions, functional indexes — `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- PostgreSQL exclusion constraints — `https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-EXCLUSION`
- PostgreSQL `btree_gist` + range overlap — `https://www.postgresql.org/docs/current/btree-gist.html` + `https://www.postgresql.org/docs/current/rangetypes.html`
- PostgreSQL partial unique indexes — `https://www.postgresql.org/docs/current/indexes-partial.html`

---

*Next artifact: implementation (plan §5 Steps 3–7) — Prisma migration (`cycle` + `issue.cycleId` + `CYCLE_CHANGED` + D3/D5/D6 raw SQL) → module code (routes/controller/service/repository + shared schemas + `assertCycleAssignable` contract) → issues-leg wiring (`cycleId` write + `?cycleId=` filter) → web slice (cycles list, detail + issue list, modals, lifecycle buttons) → tests → `pnpm check`. Dashboard current-cycle reads and full-text search land with F9/F10 via §8.7/§11.*
