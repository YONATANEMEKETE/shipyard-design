# Projects — API Design

**Status:** Draft for review
**Last updated:** 2026-08-31
**Sources:** `features/projects/spec.md` · `features/projects/data-model.md` (locked — `project` + generic `view_preference`) · `features/workspace/api-design.md` (F2 precedent — `:slug` context, guard chain, archived matrix, error envelope) · `features/members/api-design.md` (F3 precedent — RBAC pipeline, `transferOwnedProjects` contract, `confirm: true` convention) · `features/auth/api-design.md` (F1 — Better Auth session, `emailVerified`) · `00-architecture.md` §5–§8 · `ADR-001`–`ADR-003` · `Implementation Plan.md` F4

> **Principle:** identical to members (F3) — every route is hand-written Shipyard code through the canonical pipeline:
>
> ```text
> route → validation → permission check → controller → service → repository → Prisma
> ```
>
> Better Auth handles identity; this module owns **authorization** for project data — who may read projects (any member) and who may create/edit/lifecycle them (Owner/Admin). No new auth primitive; it reuses the F2/F3 guard chain verbatim.

---

## 1. Base path & conventions

| Concern | Choice |
|---|---|
| Base path | `/api/v1/workspaces/:slug/projects` and `/api/v1/workspaces/:slug/projects/:projectId` — mirrors members; `:slug` is the F2 immutable workspace token; `:projectId` is the project's `cuid()` (data-model D5 — no project slug). View preference lives at `/api/v1/workspaces/:slug/view-preferences/:scope` (generic, shared with Issues F5). |
| Next.js proxy | Browser never hits the API directly (ADR-003); `apps/web` forwards `/api/v1/*` → `http://api:4000/api/v1/*`, cookies forwarded. |
| Auth transport | HttpOnly Better Auth session cookie read by `requireSession` (F1) — `req.session.userId` is the only identity input. |
| Validation | Zod schemas from `packages/shared` (`data-model.md` §4) at the route boundary. |
| Envelope | Success: resource JSON directly (or `{ projects: [...] }` for collections). Failure: `{ "error": { "code", "message", "details"? } }` via the global error handler. |
| Workspace context | Reuses F2 `resolveWorkspaceContext(:slug)` verbatim — one authoritative resolution per request, leak-free `404 WORKSPACE_NOT_FOUND`. |
| Archived enforcement (workspace) | Mutating routes use `resolveWorkspaceContext({ rejectArchived: true })`; `GET` routes pass `rejectArchived: false`. |
| Project-level read-only (archived project) | Enforced in the **service** against `project.archivedAt` (§6.2) — restore/delete remain allowed on archived projects; everything else is rejected. |

---

## 2. Endpoint inventory

Eleven endpoints cover every behavior in `spec.md` §2–§5 and `data-model.md` §6. Project↔issue grouping and progress are **issue-owned (F5)** and are intentionally not endpoints here (§7). No extras.

### 2.1 Workspace-scoped — project CRUD & lifecycle

All under `/api/v1/workspaces/:slug/projects...`, all through the §4 guard chain.

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 1 | `GET` | `/api/v1/workspaces/:slug/projects` | `requireSession` → member (any role) | §2/§3.5 List non-archived projects (List + Kanban both derive from this one collection). Query: filters `status`, `ownerId`, `startDate`, `targetDate`; `sort`/`order`; `?archived=true` returns archived only (Archived view). Empty columns visible client-side via grouping (server never prunes a status). |
| 2 | `GET` | `/api/v1/workspaces/:slug/projects/:projectId` | `requireSession` → member (any) | §4 Detail — info, owner, progress (empty until F5), dates, description. Returns archived projects too (for the archived detail view). Scoped: `:projectId` validated against `:slug`. |
| 3 | `POST` | `/api/v1/workspaces/:slug/projects` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | §3.1 Create — creator becomes Owner, `status = ACTIVE`. Body `createProjectSchema`. |
| 4 | `PATCH` | `/api/v1/workspaces/:slug/projects/:projectId` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | §3.2/§3.5 Edit fields **and** change operational status (the non-drag status control and the Kanban cross-column drag both land here as status changes). Body `updateProjectSchema`. Rejected when project is archived (§6.2). |
| 5 | `POST` | `/api/v1/workspaces/:slug/projects/:projectId/transfer-owner` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | §3.3 Transfer Project Owner to any current member. Body `{ targetMemberId }`. Rejected when archived (spec §3.3 "transfer a non-archived project"). |
| 6 | `POST` | `/api/v1/workspaces/:slug/projects/:projectId/archive` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | §3.2 Archive — confirmed; sets `archivedAt`, keeps operational `status`. Rejected when already archived. |
| 7 | `POST` | `/api/v1/workspaces/:slug/projects/:projectId/restore` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | §3.2 Restore — confirmed; clears `archivedAt`, returns to stored operational `status`. Rejected when not archived. |
| 8 | `DELETE` | `/api/v1/workspaces/:slug/projects/:projectId` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | §3.2/rule 9 Permanent delete — verbose warning body `{ confirmName }`. Atomic unassign of issues (F5). Allowed whether archived or not. |

### 2.2 Workspace-scoped — view preference (generic, shared with Issues)

The `view_preference` table (data-model §2.2) is generic and keyed by `scope`. F4 owns the `PROJECT` scope and the generic endpoints; F5 reuses them with `scope=ISSUE` — no new endpoints needed.

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 9 | `GET` | `/api/v1/workspaces/:slug/view-preferences/:scope` | `requireSession` → member (any) | §3.5 Get the caller's view for `scope` (`PROJECT` now; `ISSUE` in F5). No row ⇒ `{ view: "LIST" }` (default, rule 12). |
| 10 | `PUT` | `/api/v1/workspaces/:slug/view-preferences/:scope` | `requireSession` → member (any) | §3.5 Upsert the caller's view for `scope`, body `{ view: "LIST"\|"KANBAN" }`. One row per `(workspace, user, scope)` (rule 12 independence — toggling projects never touches an issue row and vice-versa). |

> **Why one list endpoint serves both views:** List and Kanban show the same matching non-archived projects; they differ only in presentation (grouped sections vs columns). The client groups the fetched cards by `status` for the board; empty columns stay visible because grouping is client-side and never prunes a status. Splitting into `/list` and `/board` endpoints would duplicate the filter/sort logic and drift. The view *preference* (which presentation the user chose) is separate and lives in #9/#10.

---

## 3. Context resolution

### 3.1 Workspace-scoped routes (#1–#8) — reuse F2's resolver

Identical to members (§3.1): `resolveWorkspaceContext(:slug)` resolves the workspace + membership of `req.session.userId` in one query and attaches `req.workspaceContext = { workspaceId, slug, status, role, memberId, userId }`.

- No workspace with slug **or** no membership ⇒ generic `404 WORKSPACE_NOT_FOUND` (no existence leak).
- Membership exists, role insufficient (Member on a write route) ⇒ `403 FORBIDDEN_ROLE`.
- Workspace `ARCHIVED` + `rejectArchived: true` ⇒ `409 WORKSPACE_ARCHIVED`.
- The module-owned `:projectId` lookup then scopes to `workspaceId` (never cross-workspace) ⇒ `404 PROJECT_NOT_FOUND`.

### 3.2 View-preference routes (#9–#10) — same chain, no service beyond the row

`resolveWorkspaceContext(:slug)` (member, any role) then a direct `view_preference` read/upsert scoped to `(workspaceId, userId, scope)`. `:scope` is validated against the `ViewScope` enum at the route boundary ⇒ `400 VALIDATION_ERROR` on an unknown scope.

### 3.3 Project detail resolution (#2, #4–#8)

```text
req.params.projectId ──findFirst(where: { id, workspaceId: context.workspaceId })──▶ project
```
- No row in this workspace ⇒ `404 PROJECT_NOT_FOUND` (generic; a project id from another workspace is indistinguishable).
- The project row carries `archivedAt` — service uses it for the project-level read-only matrix (§6.2). Never resolved against a different `workspaceId`.

---

## 4. Guard chain (canonical, mirrors members §4)

### 4.1 Workspace-scoped chain (#1–#8)

```text
requireSession                     ← F1: valid Better Auth session else 401
  │
resolveWorkspaceContext(slug)      ← F2 shared middleware
  │                                  404 generic on miss/non-membership (no leak)
  │                                  409 WORKSPACE_ARCHIVED when rejectArchived && ARCHIVED
  ├─ requireWorkspaceRole("OWNER","ADMIN")   ← write routes #3–#8
  │                                            (view routes #1–#2 accept any member)
  │
projectById :projectId             ← module lookup scoped to workspaceId → 404 PROJECT_NOT_FOUND
  │
service preconditions              ← project-level matrix (§6.2): archived read-only unless
                                       restore/delete; confirm:true; name uniqueness; etc.
                                       — inside the same transaction as writes
```

Rules reaffirmed (inherited from F3): URL carries workspace context, no hidden server state; membership resolved once; role/state checks in named guards or service preconditions, never inline ad-hoc controller queries; workspace-archived workspaces read-only at the guard layer; archived-project read-only reasserted in the service.

### 4.2 View-preference chain (#9–#10)

```text
requireSession → resolveWorkspaceContext(slug, any member) → :scope enum check → read/upsert row
```
No role beyond member (view is a per-user preference). Not rejected in an archived workspace (read-only preference, and a frozen workspace may still be viewed).

---

## 5. Request/response contracts

Schemas from `packages/shared` (`data-model.md` §4). Route handlers validate bodies, params, and query before anything else.

### 5.1 Projects

| Endpoint | Body / Query | Success response |
|---|---|---|
| #1 list | query `?status=PLANNED\|ACTIVE\|COMPLETED` · `?ownerId=<userId>` · `?startDate=YYYY-MM-DD` · `?targetDate=YYYY-MM-DD` · `?sort=createdAt\|name\|targetDate\|startDate\|status` (default `createdAt`) · `?order=asc\|desc` (default `desc`) · `?archived=true` (default false) | `200` + `{ projects: projectCardSchema[] }` sorted per sort/order within a stable group-by-status for the board |
| #2 detail | — | `200` + `projectDetailSchema` |
| #3 create | `createProjectSchema` `{ name (required), description?, startDate?, targetDate? }` | `201` + `projectDetailSchema` (owner = caller; `status = ACTIVE`) |
| #4 update | `updateProjectSchema` `{ name?, description?, status?, startDate?, targetDate? }` | `200` + `projectDetailSchema` (updated) |
| #5 transfer-owner | `transferProjectOwnerSchema` `{ targetMemberId }` | `200` + `projectCardSchema` (owner updated; recipient workspace role untouched) |
| #6 archive | `{ confirm: true }` | `200` + `projectDetailSchema` (`archivedAt` set, `status` unchanged) |
| #7 restore | `{ confirm: true }` | `200` + `projectDetailSchema` (`archivedAt` null, prior `status` intact) |
| #8 delete | body `{ confirmName: string }` — echoes exact current name | `200` + `{ deletedProjectId, unassignedIssues: number }` — `unassignedIssues` is `0` until F5 wires the unassign leg (data-model §6.4/§7) |

Validation & precondition details:
- `#3`/`#4`: `name` trimmed by Zod then uniqueness re-checked at the service against the D3 `lower(name)` functional index — duplicate ⇒ `409 PROJECT_NAME_CONFLICT`. `#4` renames must also pass uniqueness (archived projects reserve the name, so a rename colliding with an archived project is still a conflict).
- `#4` `status` set here is the free-switch operational status; this endpoint is what the status control and the Kanban cross-column drag both call (drag = immediate `PATCH { status }`, no confirmation per §3.5).
- `#5` `targetMemberId` (membership row id, per F3 convention) must resolve to a current member of the same workspace; recipient may be any role; cannot be the current owner / self. Reject when project `archivedAt` set.
- `#6`/`#7` require literal `confirm: true` (missing ⇒ `400 CONFIRMATION_REQUIRED`, same precedent as workspace/members).
- `#8` requires `confirmName` to equal the exact current project name (verbose warning, spec §3.2); body required, not just `confirm: true`.
- `#4`/`#5`/`#6` reject when `archivedAt` set ⇒ `409 PROJECT_ARCHIVED` (§6.2); `#7` rejects when not archived ⇒ `409 NOT_ARCHIVED`; `#6` rejects when already archived ⇒ `409 ALREADY_ARCHIVED`.

### 5.2 View preference

| Endpoint | Body / Param | Success response |
|---|---|---|
| #9 get | `:scope` ∈ `{PROJECT}` (F5: `ISSUE`) | `200` + `{ view: "LIST"\|"KANBAN" }` — `LIST` when no row |
| #10 set | `setViewPreferenceSchema` `{ scope, view }` | `200` + `{ view }` (upserted) |

---

## 6. Read-only / archived enforcement matrices

### 6.1 Workspace-level (`workspace.status = ARCHIVED`)

| Endpoint | While ARCHIVED | Rationale |
|---|---|---|
| #1 list, #2 detail | ✅ allowed | Read-only — the frozen workspace stays browsable |
| #9 get, #10 set view pref | ✅ allowed | Preference read; set is harmless metadata, not project data mutation |
| #3–#8 all writes | ❌ `409 WORKSPACE_ARCHIVED` | No project edits in a frozen container |

Enforced at the guard layer (`rejectArchived: true`) for #3–#8.

### 6.2 Project-level (`project.archivedAt` set — own lifecycle, active workspace)

Archived **projects** are read-only (spec §3.2) but the **workspace** is active — two independent axes. Enforced in the service:

| Endpoint | While project archived | Notes |
|---|---|---|
| #1 list, #2 detail | ✅ allowed (`#1` only in the archived view via `?archived=true`) | Archived never shown on boards/lists by default |
| #4 update, #5 transfer-owner | ❌ `409 PROJECT_ARCHIVED` | Data/status/owner immutable once archived |
| #6 archive | ❌ `409 ALREADY_ARCHIVED` | Already archived |
| #7 restore | ✅ (allowed — this is the way out) | Requires `archivedAt` set; `NOT_ARCHIVED` if not |
| #8 delete | ✅ (allowed — both archived and active deletable) | Delete is not blocked by archive; it's permanent regardless |

Defense in depth: service reasserts `archivedAt` even though the guard already ran.

---

## 7. Error codes (Projects module)

Global error handler converts typed domain errors; controllers never build envelopes by hand.

| Code | HTTP | When | Notes |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | Zod body/param/query failure (`:scope` unknown, bad date, bad enum, bad sort, missing body) | `details` lists field paths |
| `CONFIRMATION_REQUIRED` | 400 | #6/#7 without literal `confirm: true` | Same precedent as workspace/members |
| `CONFIRM_NAME_MISMATCH` | 400 | #8 `confirmName` does not equal the current project name | Typed-name confirmation (spec §3.2) |
| `WORKSPACE_NOT_FOUND` | 404 | Unknown `:slug` or caller not a member — deliberately identical | No existence leak (§3.1) |
| `PROJECT_NOT_FOUND` | 404 | `:projectId` not in this workspace | Scoped — not a cross-workspace leak |
| `FORBIDDEN_ROLE` | 403 | Member on a write route (#3–#8) | Matters from F4; tested with seeded roles |
| `PROJECT_NAME_CONFLICT` | 409 | #3/#4 name normalized (trim, case-insensitive) collides with another project in the workspace, incl. archived | D3 functional index + friendly service pre-check |
| `PROJECT_ARCHIVED` | 409 | #4/#5 target is an archived project | Read-only (spec §3.2) |
| `ALREADY_ARCHIVED` | 409 | #6 on an already-archived project | |
| `NOT_ARCHIVED` | 409 | #7 on a non-archived project | |
| `TRANSFER_TARGET_INVALID` | 409 | #5 target not a current member of the workspace, is the current owner, or is self | Liveness rechecked in transaction |
| `WORKSPACE_ARCHIVED` | 409 | Mutating op while the workspace is `ARCHIVED` (§6.1) | Restorable via workspace restore |
| `UNAUTHENTICATED` | 401 | Missing/expired session cookie | F1 `requireSession` |
| `RATE_LIMITED` | 429 | Per-route create/transfer limits (wiring finalized at F12; global limiter exists) | `Retry-After` header |

---

## 8. Sequences

### 8.1 Create project (spec §3.1)

```text
Owner/Admin → POST /api/v1/workspaces/:slug/projects {name:"Ship Payroll", targetDate:"2026-12-01"}
→ requireSession ✓ → resolveWorkspaceContext ✓ (OWNER|ADMIN) → Zod validate
→ service tx {
     normalize name (trim) → assert no PROJECT_NAME_CONFLICT in workspace (lower(name))
     insert project { workspaceId, name, status: ACTIVE, ownerId: caller.userId, ... }
   } → 201 projectDetail (owner=caller)
→ client redirects to /w/:slug/projects/:id (overview)
```

### 8.2 Board drag = status change (spec §3.5)

```text
Member/Admin... (editor: Owner/Admin) drags card Planned → Active on Kanban
→ PATCH /api/v1/workspaces/:slug/projects/:id {status:"ACTIVE"}   // drop = immediate, no confirm
→ requireSession → resolveWorkspaceContext (OWNER|ADMIN, rejectArchived) → projectById
→ service asserts project.archivedAt IS NULL  → UPDATE status="ACTIVE" → 200 card
→ failure → 4xx/5xx → client returns card to previous column + error
→ concurrent change → server returns latest row; client refreshes card + notice
   (same-column drag ⇒ no reorder; card follows active sort — client holds sort, server stores none)
```

### 8.3 Transfer Project Owner (spec §3.3)

```text
Owner/Admin → POST .../projects/:id/transfer-owner {targetMemberId}
→ guard: OWNER|ADMIN, rejectArchived, project not archived
→ service tx {
     re-read project FOR UPDATE; assert archivedAt IS NULL
     resolve targetMemberId → workspace_member in same workspace (else 409 TRANSFER_TARGET_INVALID)
     assert target.userId !== project.ownerId (else conflict)
     UPDATE project SET ownerId = target.userId
   } → 200 card (recipient's workspace role untouched — ownership grants no permissions, rule 3)
```

### 8.4 Archive → restore (spec §3.2)

```text
Owner/Admin → POST .../projects/:id/archive {confirm:true}
→ guard ✓ → service: assert archivedAt IS NULL → UPDATE archivedAt=now() (status untouched) → 200

Owner/Admin → POST .../projects/:id/restore {confirm:true}
→ service: assert archivedAt IS NOT NULL → UPDATE archivedAt=NULL → 200 (returns to prior status)
→ archived project disappears from boards/lists (filter archivedAt IS NULL); visible only in Archived view
```

### 8.5 Delete (permanent, issues survive — spec rule 9)

```text
Owner/Admin → DELETE .../projects/:id  body {confirmName:"Ship Payroll"}
→ guard ✓ → service tx {
     re-read project; assert confirmName === project.name else 400 CONFIRM_NAME_MISMATCH
     const count = F5 leg: UPDATE issue SET projectId=NULL WHERE projectId=:id   // 0 until F5
     DELETE FROM project WHERE id=:id
   } → 200 { deletedProjectId, unassignedIssues: count }
→ any failure → full rollback; project and issues unchanged (all-or-nothing)
→ name released for reuse (functional index over non-deleted rows)
```

### 8.6 View preference get/set (rule 12)

```text
GET  /api/v1/workspaces/:slug/view-preferences/PROJECT → { view:"KANBAN" }   // row exists
GET  .../PROJECT for new user                        → { view:"LIST" }       // no row ⇒ default
PUT  .../PROJECT { view:"KANBAN" } → upsert row (workspaceId,userId,PROJECT) → { view:"KANBAN" }
// F5: same endpoints with :scope=ISSUE; PROJECT rows untouched by issue toggles
```

### 8.7 Leave / remove member (F3 Checkpoint B ↔ F4, for completeness)

Reuses the F3 `transferOwnedProjects` contract — no new project route:

```text
Owner/Admin removes member → members service tx {
  projectsService.transferOwnedProjects(workspaceId, fromUserId, toOwnerUserId, tx)
    // UPDATE project SET ownerId=toOwner WHERE workspaceId=? AND ownerId=from  (archived included)
  delete workspace_member ... 
}
```

---

## 9. Module layout

### 9.1 API — `apps/api/src/features/projects/`

```text
features/projects/
├── routes.ts        # router: path defs → middleware chain → controller; Zod validated at entry
├── schemas.ts       # route-local param/query coercion (dates, sort, scope); shared schemas live in packages/shared
├── controller.ts    # HTTP concerns only: parse req/query, call service, map result/errors
├── service.ts       # business rules: name uniqueness, owner transfer, archive/restore/delete,
│                    # project-level read-only matrix, view-preference upsert; transactions
├── repository.ts    # Prisma access only
└── errors.ts        # typed domain errors → global handler maps to §7
```

Shared guards reused (not owned by this module):

```text
common/guards/
├── require-session.ts           # (F1)
├── workspace-context.ts         # (F2) resolveWorkspaceContext(:slug)
└── require-workspace-role.ts    # (F2) role check — reused as requireWorkspaceRole("OWNER","ADMIN")
```

### 9.2 Shared — `packages/shared/src/projects/`

Re-exports from `data-model.md` §4 — the canonical place:

- Enums: `projectStatusSchema`, `viewScopeSchema`, `viewTypeSchema`, `projectNameSchema`
- Request: `createProjectSchema`, `updateProjectSchema`, `transferProjectOwnerSchema`, `setViewPreferenceSchema`
- Response: `projectCardSchema`, `projectDetailSchema`, `projectOwnerCardSchema`, `viewPreferenceSchema`

### 9.3 Web — `apps/web`

| Surface | Route | Reads/Writes |
|---|---|---|
| Projects page (List/Kanban + toolbar) | `/w/:slug/projects` | #1 list (filters/sort), #9 get + #10 set view pref, #3 create (from page/global menu) |
| Project details | `/w/:slug/projects/:projectId` | #2 detail, #4 update, #5 transfer, #6 archive, #7 restore, #8 delete |
| Modals | over details / page | Create, Edit, Change Owner (transfer), Archive confirm, Restore confirm, Delete confirm (shows `unassignedIssues`) |
| Global create menu | App shell | Create Project (Owner/Admin only, permission-filtered — F6 Dashboard config) |
| Archived projects | `/w/:slug/projects?archived=true` | #1 with `archived=true`, #7 restore from here |

Data access via TanStack Query hooks (like members), mutations pessimistic for create/edit/transfer/archive/restore/delete (authoritative — no optimistic status flips). Board drags are pessimistic PATCH wrapped in an optimistic-lite "pending" state; on failure the card returns to the previous column.

All surfaces ship with loading, error, empty (no projects; no projects matching filter), and permission-aware states (write affordances hidden for `MEMBER`; create/transfer/archive/delete/restore gated to Owner/Admin).

---

## 10. Testing strategy

Three layers (mirrors members §10). Tooling provisioned by F1/F2; no new deps.

### 10.1 API integration tests

Supertest against `createApp()`, real Postgres via Testcontainers + migrations. Seeded helpers: `createVerifiedUser`, `createWorkspaceAs(owner)`, `addMember(workspace, user, role)`, `createProject(workspace, overrides)`.

| Case | Covered by |
|---|---|
| Happy paths ×10 endpoints | Supertest suite per group (projects, view-preference) |
| Invalid input (bad name, bad date, bad enum, unknown scope, bad sort) | `400 VALIDATION_ERROR` |
| Missing `confirm: true` (#6/#7) | `400 CONFIRMATION_REQUIRED` |
| Delete wrong typed name (#8) | `400 CONFIRM_NAME_MISMATCH` |
| Unauthenticated ×10 | `401 UNAUTHENTICATED` |
| Non-member access (real slug, foreign user) | `404 WORKSPACE_NOT_FOUND` — assert byte-equal to unknown-slug (leak test) |
| Member on write route | `403 FORBIDDEN_ROLE` (#3–#8) |
| Create → creator is owner, status ACTIVE | `201` + DB assertions |
| Rename collision (case-insensitive, incl. archived) | `409 PROJECT_NAME_CONFLICT` |
| Duplicate create after delete | succeeds (name freed) |
| Status free-switch PLAN/ACTIVE/COMPLETED any direction | `200` per transition |
| Status on archived project | `409 PROJECT_ARCHIVED` |
| Transfer → Owner/Admin to any member, role untouched | `200` + recipient workspace role unchanged |
| Transfer → invalid target (non-member / self / current owner) | `409 TRANSFER_TARGET_INVALID` |
| Transfer → archived project | `409 PROJECT_ARCHIVED` |
| Archive → restore round trip preserves status | `200`, `archivedAt` set/cleared, `status` unchanged |
| Archive already archived / restore not archived | `409 ALREADY_ARCHIVED` / `409 NOT_ARCHIVED` |
| Delete — issues survive, assignment cleared (F5 hook, count asserted) | `200` + `unassignedIssues` count (0 pre-F5, real post-F5) + row gone + name reusable |
| Delete rollback on unassign failure (F5 hook) | project still present |
| Archived workspace writes (#3–#8) | `409 WORKSPACE_ARCHIVED` |
| Archived workspace reads (#1,#2,#9,#10) | `200` |
| Cross-workspace — project id from another workspace | `404 PROJECT_NOT_FOUND` scoped |
| View preference — no row ⇒ LIST, set ⇒ upsert, get returns it, per scope independent | `200` + DB assertions |
| List — filters status/owner/date, sort/order, archived flag | returned set matches |

### 10.2 Component tests (web) — MSW mocks of `/api/v1/workspaces/:slug/projects*` + `.../view-preferences/*`

| Surface | Cases |
|---|---|
| Projects page | Renders `projectCardSchema` list; empty state (no projects / no filter matches); Kanban groups by status with empty columns visible; List shows Planned/Active/Completed sections |
| View toggle | Segmented control calls `PUT #10`; persists; reload honors saved view; independent of issue view (future) |
| Project card | Shows name, owner, target date, progress placeholder (before F5) |
| Permission-aware toolbar | Create button hidden for `MEMBER`; status/owner/edit/archive/delete controls hidden for `MEMBER` |
| Create modal | Sends `POST #3` (MSW spy asserts schema-valid body); `PROJECT_NAME_CONFLICT` shows inline error; success navigates to detail |
| Edit modal | Sends `PATCH #4`; rename conflict shows inline; date inputs validated as `YYYY-MM-DD` |
| Transfer dialog | Owner/Admin picks member; sends `POST #5`; success updates owner; `TRANSFER_TARGET_INVALID` shows retry |
| Archive/restore confirm | Sends `POST #6`/`#7` with `confirm:true`; archived card leaves board; restore from archived view returns it |
| Delete dialog | Shows `unassignedIssues`; typed-name input required; mismatch shows error; success removes card |
| Board drag | Drop issues `PATCH #4 {status}`; failure returns card to prior column + error; concurrent change refreshes latest |
| Error envelope rendering | Every surface renders MSW-served `{error:{code,message}}` as friendly states, never raw dumps |
| Archived workspace wrapper | All mutating affordances disabled with "workspace archived" messaging |

Rules: components never re-implement business rules (e.g., "Member cannot edit" is API-enforced; web just hides controls). Tests assert wire behavior + rendered state.

### 10.3 End-to-end journey — golden path

Playwright against the composed stack (web + api + Postgres, reset between runs).

**Journey — project lifecycle (core)**

```text
1. Owner + a Member exist in a workspace (F3)
2. Owner creates "Ship Payroll" → lands on detail
3. Owner adds target date + description (PATCH) → detail updates
4. Owner drags card Planned→Active on the board (or uses status control) → status flips
5. Owner transfers ownership to the Member → Member now shown as owner
6. Owner archives it → gone from board; shown under Archived view
7. Owner restores it → back with status Active
8. Member (not owner, not admin) opens projects page → sees it, but no edit/create controls
9. Owner deletes a project with typed name → project removed
```

**Negative E2E checks (cheap):**

- **Rename conflict:** create "Payroll" then try to create "payroll" → 409 PROJECT_NAME_CONFLICT shown inline.
- **Member write attempt:** Member sends `POST #3`/`DELETE` directly → 403 FORBIDDEN_ROLE.
- **Cross-workspace leak:** second workspace's project id under first workspace's slug → 404 PROJECT_NOT_FOUND.
- **Archived project frozen:** archive then attempt `PATCH #4`/transfer → 409 PROJECT_ARCHIVED; restore then edit succeeds.
- **Archived workspace freeze:** archive the workspace, then attempt create/update/delete → 409 WORKSPACE_ARCHIVED; page still readable.

Scope discipline: journey + negatives are the mandatory F4 E2E suite; exhaustive cases stay in 10.1–10.2. Issue grouping/progress E2E lands after F5.

---

## 11. Cross-cutting concerns

| Concern | Approach |
|---|---|
| **Rate limiting** | Per-route create (10/min per workspace), transfer (10/min), delete (5/min). Memory for MVP; global limiter exists; wiring finalized at F12. |
| **Validation encoding** | Dates are `YYYY-MM-DD` strings end-to-end (data-model D4, `@db.Date`); api coerces to `Date`, web formats from the same schema. |
| **Sorting / ordering** | Server applies `sort`/`order`; board groups by `status` client-side and applies the sort within columns (no manual ranking in MVP, spec §3.5). Server never stores order. |
| **Pagination** | None — projects are few (spec Q2). A `LIMIT` cap at the API layer for safety, not structural pagination. |
| **Audit / activity** | No audit table in F4 (spec Q1 open). `project.createdAt`/`updatedAt` cover record-level display; action-level activity deferred with Issues. |
| **Progress / grouping** | Derived from issues (F5) — see §7. No endpoint returns progress in F4; the card schema intentionally has no progress field yet (data-model §7). |
| **Search** | Project board/list filtering is exact-match query params in F4; global search lands with F10 Search (Postgres full-text on `project.name`). |

---

## 12. References

- Shipyard: `features/projects/spec.md`, `features/projects/data-model.md`, `features/workspace/api-design.md` (guard chain, archived matrix, envelope), `features/members/api-design.md` (RBAC pipeline, `transferOwnedProjects`, `confirm: true`), `features/auth/api-design.md` (session, `emailVerified`), `00-architecture.md` §5–§8, `ADR-001`–`ADR-003`, `Implementation Plan.md` F4
- Prisma: referential actions, functional indexes — `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- PostgreSQL functional index — `https://www.postgresql.org/docs/current/indexes-expressional.html`

---

*Next artifact: implementation (plan §5 Steps 3–7) — Prisma migration → module code (routes/controller/service/repository + shared schemas) → web slice (projects list/board, detail, modals, view toggle) → tests → `pnpm check`. Issue grouping/progress/unassign legs land with F5 and are called out in §7.*
