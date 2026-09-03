# Issues — API Design

**Status:** Draft for review
**Last updated:** 2026-09-03
**Sources:** `features/issues/spec.md` · `features/issues/data-model.md` (locked — `issue` + `label` + `issue_label` + `issue_history` + `workspace_issue_sequence`, `ViewScope += ISSUE`) · `features/workspace/api-design.md` (F2 precedent — `:slug` context, guard chain, archived matrix, error envelope) · `features/members/api-design.md` (F3 precedent — RBAC pipeline, `confirm: true` convention, unassign contract) · `features/projects/api-design.md` (F4 precedent — archive dimension, board-drag-as-PATCH, view-preference reuse, `SetNull` unassign leg) · `features/auth/api-design.md` (F1 — Better Auth session, `emailVerified`) · `00-architecture.md` §5–§8 · `ADR-001`–`ADR-003` · `Implementation Plan.md` F5

> **Principle:** identical to projects (F4) — every route is hand-written Shipyard code through the canonical pipeline:
>
> ```text
> route → validation → permission check → controller → service → repository → Prisma
> ```
>
> Better Auth handles identity; this module owns **authorization** for issue data — who may read/write issues (any member) and who may permanently delete them (Owner/Admin only). No new auth primitive; it reuses the F2/F3 guard chain verbatim.

---

## 1. Base path & conventions

| Concern | Choice |
|---|---|
| Base path | `/api/v1/workspaces/:slug/issues` and `/api/v1/workspaces/:slug/issues/:issueId` — mirrors projects; `:slug` is the F2 immutable workspace token; `:issueId` is the issue's `cuid()` (never the `SHIP-###` display identifier in URLs — identifiers are display-only, `id` is the reference key, same reasoning as projects D5). Labels live at `/api/v1/workspaces/:slug/labels...`; attach/detach nests under the issue. History reads at `.../issues/:issueId/history`. |
| Next.js proxy | Browser never hits the API directly (ADR-003); `apps/web` forwards `/api/v1/*` → `http://api:4000/api/v1/*`, cookies forwarded. |
| Auth transport | HttpOnly Better Auth session cookie read by `requireSession` (F1) — `req.session.userId` is the only identity input. |
| Validation | Zod schemas from `packages/shared` (`data-model.md` §4) at the route boundary. |
| Envelope | Success: resource JSON directly (or `{ issues: [...], nextCursor }` for collections). Failure: `{ "error": { "code", "message", "details"? } }` via the global error handler. |
| Workspace context | Reuses F2 `resolveWorkspaceContext(:slug)` verbatim — one authoritative resolution per request, leak-free `404 WORKSPACE_NOT_FOUND`. |
| Archived enforcement (workspace) | Mutating routes use `resolveWorkspaceContext({ rejectArchived: true })`; `GET` routes pass `rejectArchived: false`. |
| Issue-level read-only (archived issue) | Enforced in the **service** against `issue.archivedAt` (§6.2) — restore/delete/history-read remain allowed on archived issues; everything else is rejected. |

---

## 2. Endpoint inventory

Fourteen endpoints cover every behavior in `spec.md` §2–§5 and `data-model.md` §6. Cycle assignment, notifications emission, comments, full-text search, and dashboard aggregation are **not** endpoints here (§7). View preference reuses the F4 generic endpoints with `scope=ISSUE` — no new endpoints. No extras.

### 2.1 Workspace-scoped — issue CRUD & lifecycle

All under `/api/v1/workspaces/:slug/issues...`, all through the §4 guard chain.

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 1 | `GET` | `/api/v1/workspaces/:slug/issues` | `requireSession` → member (any role) | §2/§3.5 List non-archived issues (List + Kanban both derive from this one collection). Query: filters `status`, `priority`, `assigneeId`, `projectId`, `labels` (comma-separated label ids, AND semantics), `blocked`, `dueDateFrom/To`, `q` (F5 basic search); `sort`/`order`; cursor `cursor` + `limit`; `?archived=true` returns archived only (Archived view). Subview mappings in §5.1. Empty columns visible client-side via grouping (server never prunes a status). |
| 2 | `GET` | `/api/v1/workspaces/:slug/issues/:issueId` | `requireSession` → member (any) | §4 Detail — full card + description + creator + labels. Returns archived issues too (for the archived detail view). Scoped: `:issueId` validated against `:slug`. |
| 3 | `POST` | `/api/v1/workspaces/:slug/issues` | `requireSession` → member (any) + `rejectArchived` | §3.1 Create — any member. Body `createIssueSchema`. Allocates `seqNumber` atomically, defaults `BACKLOG / NO_PRIORITY / unblocked`. |
| 4 | `PATCH` | `/api/v1/workspaces/:slug/issues/:issueId` | `requireSession` → member (any) + `rejectArchived` | §3.2/§3.3/§3.4 Edit everything: title/description/status/priority/assignee/project/due/blocked. The status control, Kanban cross-column drag, block toggle, assignee picker, and planning editors all land here. Body `updateIssueSchema`. Rejected when archived (§6.2). |
| 5 | `POST` | `/api/v1/workspaces/:slug/issues/:issueId/archive` | `requireSession` → member (any) + `rejectArchived` | §3.6 Archive — confirmed; sets `archivedAt`, keeps `status` + `blocked`. Rejected when already archived. |
| 6 | `POST` | `/api/v1/workspaces/:slug/issues/:issueId/restore` | `requireSession` → member (any) + `rejectArchived` | §3.6 Restore — confirmed; clears `archivedAt`, returns to stored status + blocked. Rejected when not archived. |
| 7 | `DELETE` | `/api/v1/workspaces/:slug/issues/:issueId` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | §3.6 Permanent delete — **only** Owner/Admin (spec rule 10 divergence from other issue writes). Verbose body `{ confirmIdentifier: "SHIP-24" }`. Cascades joins + history (+ F6/F8 descendants per §7). Identifier never reused. Allowed whether archived or not. |
| 8 | `GET` | `/api/v1/workspaces/:slug/issues/:issueId/history` | `requireSession` → member (any) | §2 History — chronological `issue_history` rows for the issue. Cursor paginated (`cursor` + `limit`). Allowed on archived issues (history is read-only facts). |

### 2.2 Workspace-scoped — labels (owned by this module, like view_preference was owned by projects)

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 9 | `GET` | `/api/v1/workspaces/:slug/labels` | `requireSession` → member (any) | §3.7 List all workspace labels (small set, no pagination — like projects list). Sorted by `name`. |
| 10 | `POST` | `/api/v1/workspaces/:slug/labels` | `requireSession` → member (any) + `rejectArchived` | §3.7 Create label — any member (data-model D6). Body `createLabelSchema`. |
| 11 | `PATCH` | `/api/v1/workspaces/:slug/labels/:labelId` | `requireSession` → member (any) + `rejectArchived` | §3.7 Rename/recolor. Body `updateLabelSchema`. Scoped to `:slug`. |
| 12 | `DELETE` | `/api/v1/workspaces/:slug/labels/:labelId` | `requireSession` → member (any) + `rejectArchived` | §3.7 Delete — confirmed `{ confirm: true }`; unlinks from issues via cascade (issues untouched). Any member (D6 — labels are shared vocabulary, not privileged). |
| 13 | `POST` | `/api/v1/workspaces/:slug/issues/:issueId/labels` | `requireSession` → member (any) + `rejectArchived` | Attach — body `{ labelId }`. Asserts same workspace. Emits `LABEL_ADDED`. Idempotent guard: already-attached ⇒ `409 LABEL_ALREADY_ATTACHED`. Rejected when issue archived. |
| 14 | `DELETE` | `/api/v1/workspaces/:slug/issues/:issueId/labels/:labelId` | `requireSession` → member (any) + `rejectArchived` | Detach — emits `LABEL_REMOVED`. Not-attached ⇒ `409 LABEL_NOT_ATTACHED`. Rejected when issue archived. |

> **Why one list endpoint serves both views:** List and Kanban show the same matching non-archived issues; they differ only in presentation (flat list vs four status columns). The client groups the fetched cards by `status` for the board; empty columns stay visible because grouping is client-side and never prunes a status. Splitting into `/list` and `/board` would duplicate filter/sort/search logic and drift. The view *preference* (which presentation the user chose) lives in the reused F4 endpoints `GET/PUT /view-preferences/ISSUE` — no new endpoints here.
>
> **Why blocked has no dedicated endpoint:** blocked is an orthogonal flag on the issue row (data-model D9), not a separate resource. `PATCH #4 { blocked: true, blockedReason }` / `{ blocked: false }` is the set/clear path; the Done-clears-blocked side effect rides the same `PATCH #4 { status: DONE }`.

---

## 3. Context resolution

### 3.1 Workspace-scoped routes (#1–#14) — reuse F2's resolver

Identical to projects (§3.1): `resolveWorkspaceContext(:slug)` resolves the workspace + membership of `req.session.userId` in one query and attaches `req.workspaceContext = { workspaceId, slug, status, role, memberId, userId }`.

- No workspace with slug **or** no membership ⇒ generic `404 WORKSPACE_NOT_FOUND` (no existence leak).
- Membership exists, role insufficient (Member on #7 delete) ⇒ `403 FORBIDDEN_ROLE`.
- Workspace `ARCHIVED` + `rejectArchived: true` ⇒ `409 WORKSPACE_ARCHIVED`.
- Module-owned `:issueId` / `:labelId` lookups scope to `workspaceId` (never cross-workspace) ⇒ `404 ISSUE_NOT_FOUND` / `404 LABEL_NOT_FOUND`.

### 3.2 Issue / label detail resolution (#2, #4–#8, #11–#14)

```text
req.params.issueId ──findFirst(where: { id, workspaceId: context.workspaceId })──▶ issue
req.params.labelId ──findFirst(where: { id, workspaceId: context.workspaceId })──▶ label
```

- No row in this workspace ⇒ `404` scoped (a foreign-workspace id is indistinguishable from a nonexistent one).
- The issue row carries `archivedAt` + `blocked` — service uses them for the read-only/blocked matrices (§6.2). Never resolved against a different `workspaceId`.
- Attach (#13) resolves **both** rows and asserts `label.workspaceId === issue.workspaceId === context.workspaceId` (defense in depth; both lookups are already scoped, so mismatch ⇒ `409 LABEL_NOT_IN_WORKSPACE` only if a caller smuggles ids across scopes — normally unreachable, kept for explicitness).

### 3.3 History resolution (#8)

Same issue lookup, then `issue_history` read scoped to `(workspaceId, issueId)` ordered `(createdAt ASC, id ASC)` — chronological (oldest first), matching the comments-chronology convention (F8). Cursor is opaque base64 of `(createdAt, id)`.

---

## 4. Guard chain (canonical, mirrors projects §4)

### 4.1 Issue routes (#1–#8)

```text
requireSession                     ← F1: valid Better Auth session else 401
  │
resolveWorkspaceContext(slug)      ← F2 shared middleware
  │                                  404 generic on miss/non-membership (no leak)
  │                                  409 WORKSPACE_ARCHIVED when rejectArchived && ARCHIVED
  ├─ requireWorkspaceRole("OWNER","ADMIN")   ← #7 delete ONLY
  │                                            (all other issue routes accept any member —
  │                                             deliberate divergence from projects, spec rule 10)
  │
issueById :issueId                ← module lookup scoped to workspaceId → 404 ISSUE_NOT_FOUND
  │
service preconditions              ← issue-level matrix (§6.2): archived read-only unless
                                       restore/delete/history-read; blocked preconditions;
                                       assignee/project/label same-workspace checks;
                                       confirmIdentifier gate — inside the same transaction as writes
```

### 4.2 Label routes (#9–#14)

```text
requireSession → resolveWorkspaceContext(slug, any member) → labelById (where needed)
  → service preconditions (same-workspace, name uniqueness, attach/detach guards)
```

No role beyond member on any label route (data-model D6). Not rejected beyond the standard workspace-archived freeze.

Rules reaffirmed (inherited from F2–F4): URL carries workspace context, no hidden server state; membership resolved once; role/state checks in named guards or service preconditions, never inline ad-hoc controller queries; workspace-archived workspaces read-only at the guard layer; archived-issue read-only reasserted in the service.

---

## 5. Request/response contracts

Schemas from `packages/shared` (`data-model.md` §4). Route handlers validate bodies, params, and query before anything else.

### 5.1 Issues

| Endpoint | Body / Query | Success response |
|---|---|---|
| #1 list | query `?status=BACKLOG\|TODO\|IN_PROGRESS\|DONE` (repeatable or comma-separated) · `?priority=...` (repeatable) · `?assigneeId=<userId>` (`me` alias resolves to caller) · `?projectId=<cuid>` · `?labels=<labelId,-describedby>` (comma-separated, AND semantics — issue must carry **all** listed labels) · `?blocked=true\|false` · `?dueDateFrom=YYYY-MM-DD` · `?dueDateTo=YYYY-MM-DD` · `?q=<text>` (F5 basic: `ILIKE title/description`, exact `SHIP-###` match, trimmed, max 200 chars, min 2 chars else ignored) · `?sort=createdAt\|updatedAt\|priority\|dueDate\|seqNumber` (default `createdAt`) · `?order=asc\|desc` (default `desc`, except Backlog subview default `seqNumber asc` client hint) · `?limit=1..100` (default `25`) · `?cursor=<opaque>` · `?archived=true` (default false) | `200` + `{ issues: issueCardSchema[], nextCursor: string \| null }` — `nextCursor null` ⇒ end. Cards carry `labels[]` inline (no second fetch for the board). |
| #2 detail | — | `200` + `issueDetailSchema` (card + `description` + `creator` + `labels[]`) |
| #3 create | `createIssueSchema` `{ title (required), description?, priority?, status?, assigneeId?, projectId?, labelIds?, dueDate? }` | `201` + `issueDetailSchema` (`seqNumber` allocated, `identifier SHIP-{seq}`, `status BACKLOG` / `priority NO_PRIORITY` / `blocked false` when omitted) |
| #4 update | `updateIssueSchema` `{ title?, description?, status?, priority?, assigneeId?, projectId?, dueDate?, blocked?, blockedReason? }` (`null` on nullable fields ⇒ explicitly unset; omitted ⇒ leave as is) | `200` + `issueDetailSchema` (updated). Multi-field PATCH applies atomically and emits one history row per changed concern. |
| #5 archive | `{ confirm: true }` | `200` + `issueDetailSchema` (`archivedAt` set, `status` + `blocked` unchanged) |
| #6 restore | `{ confirm: true }` | `200` + `issueDetailSchema` (`archivedAt` null, prior `status` + `blocked` intact) |
| #7 delete | body `{ confirmIdentifier: string }` — must equal the issue's exact `SHIP-###` | `200` + `{ deletedIssueId, identifier }` — joins + history (+ F6/F8 descendants) removed atomically; number never reused |
| #8 history | query `?limit=1..100` (default `50`) · `?cursor=<opaque>` | `200` + `{ history: issueHistoryCardSchema[], nextCursor: string \| null }` chronological (oldest first) |

Subview mappings (client conventions over #1 primitives — no separate endpoints):

| Subview | Query equivalent |
|---|---|
| All Issues | `archived` omitted (false) + active filters/sort/search |
| My Issues | `assigneeId=me` + active filters/sort/search |
| Backlog | `status=BACKLOG` + active filters/search |
| Blocked | `blocked=true` + active filters/search |
| Archived | `archived=true` (list-only; Kanban toggle hidden) |

Validation & precondition details:

- `#3`/`#4`: `assigneeId` (user id) must resolve to a current member of the same workspace ⇒ else `404 ASSIGNEE_NOT_MEMBER` (scoped 404, not 400 — the user exists but is not addressable here). `projectId` must resolve to a same-workspace project with `archivedAt IS NULL` ⇒ else `404 PROJECT_NOT_IN_WORKSPACE` / `409 PROJECT_ARCHIVED`. `labelIds`/`labelId` must each resolve same-workspace ⇒ else `404 LABEL_NOT_IN_WORKSPACE`.
- `#4` `status` is the free-switch workflow status; Kanban cross-column drag calls `PATCH #4 { status }` with no confirmation. Same-column drag sends nothing (client holds sort, server stores none).
- `#4` blocked: `blocked: true` requires `status ∈ {BACKLOG, TODO, IN_PROGRESS}` and `archivedAt IS NULL` ⇒ else `409 CANNOT_BLOCK_DONE` / `409 ISSUE_ARCHIVED`. `blockedReason` trimmed; empty ⇒ `null`; >500 chars ⇒ `400 VALIDATION_ERROR`. `status: DONE` on a blocked issue atomically clears `blocked/blockedReason` (no separate call needed). Explicit `blocked: false` clears reason.
- `#4` same-person assignee set is a no-op: no write, no history, no notification event (spec rule 13). Detected by comparing incoming `assigneeId` to stored (both `null` ⇒ no-op).
- `#5`/`#6` require literal `confirm: true` (missing ⇒ `400 CONFIRMATION_REQUIRED`, same precedent as workspace/members/projects).
- `#7` requires `confirmIdentifier` to equal the exact current `SHIP-###` (verbosetyped confirmation adapted from projects' `confirmName` — titles are non-unique so the identifier is the only unambiguous confirmation); mismatch ⇒ `400 CONFIRM_IDENTIFIER_MISMATCH`.
- `#4`/`#5`/`#13`/`#14` reject when `archivedAt` set ⇒ `409 ISSUE_ARCHIVED` (§6.2); `#5` rejects when already archived ⇒ `409 ALREADY_ARCHIVED`; `#6` rejects when not archived ⇒ `409 NOT_ARCHIVED`.
- `#1` `q`: trimmed server-side; <2 chars after trim ⇒ ignored (not an error — avoids single-char table scans); >200 chars ⇒ `400 VALIDATION_ERROR`. Exact-identifier match (`/^SHIP-(\d+)$/i`) short-circuits to the one issue when it exists in the workspace (still subject to `archived` flag).
- `#1` cursor: opaque base64url of the last row's sort key + `id` tiebreak. Unknown/malformed cursor ⇒ `400 VALIDATION_ERROR`. Cursor is sort-specific — changing `sort`/`order`/filters invalidates outstanding cursors (client discards and refetches from head).

### 5.2 Labels

| Endpoint | Body / Param | Success response |
|---|---|---|
| #9 list | — | `200` + `{ labels: labelCardSchema[] }` sorted by `name` |
| #10 create | `createLabelSchema` `{ name (required), color? }` | `201` + `labelCardSchema` |
| #11 update | `updateLabelSchema` `{ name?, color? }` | `200` + `labelCardSchema` |
| #12 delete | `{ confirm: true }` | `200` + `{ deletedLabelId, unlinkedIssues: number }` — joins removed, issues untouched |
| #13 attach | `{ labelId }` | `200` + `issueDetailSchema` (labels updated) |
| #14 detach | — (`:labelId` in path) | `200` + `issueDetailSchema` (labels updated) |

- `#10`/`#11`: `name` trimmed then uniqueness re-checked against the D6 `lower(name)` functional index — duplicate ⇒ `409 LABEL_NAME_CONFLICT`.
- `#12` requires literal `confirm: true` ⇒ else `400 CONFIRMATION_REQUIRED`.

### 5.3 View preference (reused, no new endpoints)

`GET/PUT /api/v1/workspaces/:slug/view-preferences/ISSUE` (F4 §5.2 with `:scope=ISSUE`): absent row ⇒ `{ view: "LIST" }`; upsert `{ view }`. Toggling issues never touches `PROJECT` rows (rule 12).

---

## 6. Read-only / archived enforcement matrices

### 6.1 Workspace-level (`workspace.status = ARCHIVED`)

| Endpoint | While ARCHIVED | Rationale |
|---|---|---|
| #1 list, #2 detail, #8 history, #9 label list | ✅ allowed | Read-only — the frozen workspace stays browsable |
| View-preference `GET/PUT ISSUE` | ✅ allowed | Harmless per-user metadata |
| #3–#7, #10–#14 all writes | ❌ `409 WORKSPACE_ARCHIVED` | No issue/label edits in a frozen container |

Enforced at the guard layer (`rejectArchived: true`) for all writes.

### 6.2 Issue-level (`issue.archivedAt` set — own lifecycle, active workspace)

Archived **issues** are read-only (spec §3.6) but the **workspace** is active — two independent axes. Enforced in the service:

| Endpoint | While issue archived | Notes |
|---|---|---|
| #1 list, #2 detail, #8 history | ✅ allowed (`#1` only in the archived view via `?archived=true`) | Archived never shown on boards/lists by default |
| #4 update, #13 attach, #14 detach | ❌ `409 ISSUE_ARCHIVED` | Data/status/labels immutable once archived |
| #11/#12 label ops targeting the archived issue's labels collection | n/a — labels are workspace entities; label rename/delete is allowed (it does not mutate the archived issue row itself; the join rows to archived issues are left intact and re-render on restore) |
| #5 archive | ❌ `409 ALREADY_ARCHIVED` | Already archived |
| #6 restore | ✅ (allowed — this is the way out) | Requires `archivedAt` set; `NOT_ARCHIVED` if not |
| #7 delete | ✅ (allowed — both archived and active deletable) | Delete is not blocked by archive; permanent regardless |

Defense in depth: service reasserts `archivedAt` even though the guard already ran.

---

## 7. Error codes (Issues module)

Global error handler converts typed domain errors; controllers never build envelopes by hand.

| Code | HTTP | When | Notes |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | Zod body/param/query failure (bad title, bad enum, bad date, bad cursor/limit, bad `q`, bad hex color) | `details` lists field paths |
| `CONFIRMATION_REQUIRED` | 400 | #5/#6/#12 without literal `confirm: true` | Same precedent as workspace/members/projects |
| `CONFIRM_IDENTIFIER_MISMATCH` | 400 | #7 `confirmIdentifier` does not equal the issue's `SHIP-###` | Titles non-unique — identifier is the confirmation key |
| `WORKSPACE_NOT_FOUND` | 404 | Unknown `:slug` or caller not a member — deliberately identical | No existence leak (§3.1) |
| `ISSUE_NOT_FOUND` | 404 | `:issueId` not in this workspace | Scoped — not a cross-workspace leak |
| `LABEL_NOT_FOUND` | 404 | `:labelId` not in this workspace | Scoped |
| `ASSIGNEE_NOT_MEMBER` | 404 | `assigneeId` is not a current member of the workspace | Scoped 404 (addressability, not input shape) |
| `PROJECT_NOT_IN_WORKSPACE` | 404 | `projectId` not in this workspace | Scoped |
| `LABEL_NOT_IN_WORKSPACE` | 404 | `labelId` not in this workspace | Scoped |
| `FORBIDDEN_ROLE` | 403 | Member on #7 delete | Only delete is role-gated in this module |
| `LABEL_NAME_CONFLICT` | 409 | #10/#11 name normalized (trim, case-insensitive) collides, incl. any label | D6 functional index + friendly pre-check |
| `ISSUE_ARCHIVED` | 409 | #4/#13/#14 target is archived | Read-only (spec §3.6) |
| `ALREADY_ARCHIVED` | 409 | #5 on an already-archived issue | |
| `NOT_ARCHIVED` | 409 | #6 on a non-archived issue | |
| `CANNOT_BLOCK_DONE` | 409 | `blocked: true` on a `DONE` issue (or `status: DONE` + `blocked: true` in one PATCH) | Only unfinished blockable (rule 6) |
| `PROJECT_ARCHIVED` | 409 | Attach to an archived project | Archived projects read-only (F4) |
| `LABEL_ALREADY_ATTACHED` | 409 | #13 label already on the issue | Idempotency guard — client treats as success-or-refresh |
| `LABEL_NOT_ATTACHED` | 409 | #14 label not on the issue | |
| `WORKSPACE_ARCHIVED` | 409 | Mutating op while the workspace is `ARCHIVED` (§6.1) | Restorable via workspace restore |
| `UNAUTHENTICATED` | 401 | Missing/expired session cookie | F1 `requireSession` |
| `RATE_LIMITED` | 429 | Per-route create/update limits (wiring finalized at F12; global limiter exists) | `Retry-After` header |

---

## 8. Sequences

### 8.1 Create issue (spec §3.1)

```text
Member → POST /api/v1/workspaces/:slug/issues {title:"Fix login redirect", priority:"HIGH", assigneeId, projectId, labelIds:[...]}
→ requireSession ✓ → resolveWorkspaceContext ✓ (any member) → Zod validate
→ service tx {
     seq = UPDATE workspace_issue_sequence SET nextNumber = nextNumber+1 RETURNING nextNumber-1  // D2 atomic
     assert assignee/project/labels same-workspace (+ project not archived)
     INSERT issue { workspaceId, seqNumber: seq, status: BACKLOG (or requested), priority, ... }
     INSERT issue_label rows
     INSERT issue_history { event: CREATED, actor: caller }
     // F6 hook: if assigneeId set → expose assignment event { issueId, newAssigneeId, actorId } (§7)
   } → 201 detail (identifier SHIP-{seq})
→ assignee notified (F6, same tx) → client appears in Backlog + board
```

### 8.2 Board drag = status change (spec §3.5)

```text
Member drags card TODO → IN_PROGRESS on Kanban
→ PATCH /api/v1/workspaces/:slug/issues/:id {status:"IN_PROGRESS"}   // drop = immediate, no confirm
→ requireSession → resolveWorkspaceContext (any member, rejectArchived) → issueById
→ service asserts archivedAt IS NULL → UPDATE status + INSERT STATUS_CHANGED {old,new} → 200 detail
→ failure → 4xx/5xx → client returns card to previous column + error
→ concurrent change → server returns latest row; client refreshes card + notice
   (same-column drag ⇒ nothing sent; card follows active sort — client holds sort, server stores none)
```

### 8.3 Block → Done clears flag (spec §3.3)

```text
Member → PATCH .../issues/:id {blocked:true, blockedReason:"Waiting on API key"}
→ guard ✓ → service: assert status ∈ active → UPDATE blocked=true + INSERT BLOCKED_SET → 200 (badge on card)

Member → PATCH .../issues/:id {status:"DONE"}
→ service tx {
     UPDATE status=DONE + INSERT STATUS_CHANGED
     // blocked was true → also UPDATE blocked=false, blockedReason=NULL + INSERT BLOCKED_CLEARED
   } → 200 (badge gone)
→ PATCH .../issues/:id {status:"TODO"} later → only STATUS_CHANGED; blocked stays cleared
```

### 8.4 Archive → restore (spec §3.6)

```text
Member → POST .../issues/:id/archive {confirm:true}
→ guard ✓ → service: assert archivedAt IS NULL → UPDATE archivedAt=now() + INSERT ARCHIVED → 200
→ gone from boards/lists (filter archivedAt IS NULL); visible only in Archived view

Member → POST .../issues/:id/restore {confirm:true}
→ service: assert archivedAt IS NOT NULL → UPDATE archivedAt=NULL + INSERT RESTORED → 200
→ back with stored status + blocked state intact
```

### 8.5 Delete (permanent — Owner/Admin only, spec rule 10)

```text
Owner/Admin → DELETE .../issues/:id  body {confirmIdentifier:"SHIP-24"}
→ guard: OWNER|ADMIN, rejectArchived, issue scoped ✓ → service tx {
     re-read issue FOR UPDATE; assert confirmIdentifier === issue.identifier else 400
     DELETE FROM issue WHERE id=:id   // cascades issue_label + issue_history (+ F6 notifications, F8 comments per §7)
   } → 200 { deletedIssueId, identifier }
→ any failure → full rollback; issue and descendants unchanged (all-or-nothing)
→ sequence untouched — SHIP-24 never reused
```

### 8.6 Label lifecycle (spec §3.7)

```text
Member → POST .../labels {name:"bug", color:"#EF4444"}
→ guard ✓ → service: trim + assert no LABEL_NAME_CONFLICT → INSERT → 201

Member → POST .../issues/:id/labels {labelId}
→ guard ✓ → service: assert same-workspace + issue not archived + not already attached
→ INSERT join + INSERT LABEL_ADDED → 200 detail

Member → DELETE .../labels/:labelId {confirm:true}
→ DELETE label → joins cascade (issues untouched) → 200 { deletedLabelId, unlinkedIssues }
```

### 8.7 Leave / remove member (F3 ↔ F5, for completeness)

Reuses the membership-exit contract — no new issue route:

```text
Owner/Admin removes member → members service tx {
  issuesService.unassignOnMemberExit(workspaceId, departingUserId, tx)
    // UPDATE issue SET assigneeId=NULL WHERE workspaceId=? AND assigneeId=?  (archived included)
    // + INSERT UNASSIGNED history per affected issue (actor = remover)
  delete workspace_member ...
}
```

### 8.8 Project delete (F4 ↔ F5, closes the F4 leg)

```text
Owner/Admin → DELETE .../projects/:id (F4 #8) → projects service tx {
  UPDATE issue SET projectId=NULL WHERE projectId=:id   // F5 leg, now live
    // + INSERT PROJECT_CHANGED {old: projectId, new: null} per affected issue
  DELETE FROM project WHERE id=:id
}
```

---

## 9. Module layout

### 9.1 API — `apps/api/src/features/issues/`

```text
features/issues/
├── routes.ts        # router: path defs → middleware chain → controller; Zod validated at entry
│                    # (#1–#8 issues, #9–#14 labels + attach/detach)
├── schemas.ts       # route-local param/query coercion (slug/issueId/labelId params,
│                    # list filters/sort/cursor/q, archived flag); shared schemas in packages/shared
├── controller.ts    # HTTP concerns only: parse req/query, call service, map result/errors
├── service.ts       # business rules: seq allocation, same-workspace checks, blocked matrix,
│                    # archive/restore/delete, label ops, history writes, unassign contract,
│                    # assignment-event exposure; transactions
├── repository.ts    # Prisma access only (issue + label + join + history + sequence queries)
└── errors.ts        # typed domain errors → global handler maps to §7
```

Labels live inside the issues module (same ownership reasoning as view_preference inside projects) — one module owns the board and everything rendered on it. Shared guards reused (not owned by this module):

```text
common/guards/
├── require-session.ts           # (F1)
├── workspace-context.ts         # (F2) resolveWorkspaceContext(:slug)
└── require-workspace-role.ts    # (F2) — used ONLY for #7 delete as requireWorkspaceRole("OWNER","ADMIN")
```

Cross-module contracts owned elsewhere, called here:

```text
membersService  → resolveWorkspaceContext, membership lookups (assignee validation, unassign call site)
projectsService → project lookups (assignment validation) + transferOwnedProjects (untouched by F5)
notificationsService (F6) → createAssignment(event) inside the issues tx (§7)
```

### 9.2 Shared — `packages/shared/src/issues/`

Re-exports from `data-model.md` §4 — the canonical place:

- Enums: `issueStatusSchema`, `issuePrioritySchema`, `issueHistoryEventSchema` (+ widened `viewScopeSchema` with `ISSUE`)
- Bounds: `issueTitleSchema`, `issueDescriptionSchema`, `blockedReasonSchema`, `labelNameSchema`, `labelColorSchema`, `issueDateSchema`
- Request: `createIssueSchema`, `updateIssueSchema`, `createLabelSchema`, `updateLabelSchema`, list-query schemas (filters/sort/cursor), attach/detach schemas, `confirmIdentifier` schema
- Response: `issueCardSchema`, `issueDetailSchema`, `labelCardSchema`, `issueHistoryCardSchema`, list/history page schemas (`{ issues, nextCursor }`)

### 9.3 Web — `apps/web`

| Surface | Route | Reads/Writes |
|---|---|---|
| Issues page (List/Kanban + toolbar + filters + search) | `/w/:slug/issues` (+ `?status=&priority=&assignee=&project=&labels=&blocked=&q=&sort=&order=`) | #1 list (filters/sort/search/cursor), view-preference `GET/PUT ISSUE`, #3 create (page/global menu/quick-add) |
| Issue detail | `/w/:slug/issues/:issueId` | #2 detail, #4 update, #5 archive, #6 restore, #7 delete, #8 history, #13/#14 label attach/detach |
| Labels manager | over issues page / detail | #9 list, #10 create, #11 rename/recolor, #12 delete |
| Subviews | same `/w/:slug/issues` with preset queries (§5.1) | All/My/Backlog/Blocked map to #1 combos; Archived maps to `?archived=true` (Kanban hidden) |
| Global create menu | App shell | Create Issue (all roles, permission-filtered — F9 Dashboard config) |

Data access via TanStack Query hooks (like members/projects): infinite-query for #1 (cursor), standard queries for #2/#8/#9, mutations pessimistic for create/update/archive/restore/delete/labels. Board drags are pessimistic `PATCH #4 {status}` wrapped in an optimistic-lite "pending" state; on failure the card returns to the previous column.

All surfaces ship with loading, error, empty (no issues; no issues matching filter; no labels; empty history), and permission-aware states (delete affordance hidden for `MEMBER`; archived-issue editors disabled with "archived — restore to edit" messaging).

---

## 10. Testing strategy

Three layers (mirrors projects §10). Tooling provisioned by F1/F2; no new deps.

### 10.1 API integration tests

Supertest against `createApp()`, real Postgres via Testcontainers + migrations. Seeded helpers: `createVerifiedUser`, `createWorkspaceAs(owner)`, `addMember(workspace, user, role)`, `createProject(workspace, overrides)`, `createLabel(workspace, overrides)`, `createIssue(workspace, overrides)`.

| Case | Covered by |
|---|---|
| Happy paths ×14 endpoints | Supertest suite per group (issues, labels, history) |
| Invalid input (empty title, overlong title/desc/reason, bad enum/date/color/cursor/limit/sort, bad `q`) | `400 VALIDATION_ERROR` |
| Missing `confirm: true` (#5/#6/#12) | `400 CONFIRMATION_REQUIRED` |
| Delete wrong identifier (#7) | `400 CONFIRM_IDENTIFIER_MISMATCH` |
| Unauthenticated ×14 | `401 UNAUTHENTICATED` |
| Non-member access (real slug, foreign user) | `404 WORKSPACE_NOT_FOUND` — assert byte-equal to unknown-slug (leak test) |
| Member on #7 delete | `403 FORBIDDEN_ROLE` (Member can do every other write — assert 200s) |
| Create → defaults Backlog/No Priority/unblocked + `SHIP-{seq}` | `201` + DB assertions |
| Concurrent creates → distinct sequential `seqNumber`s | parallel POSTs, assert uniqueness + monotonicity |
| Delete → identifier never reused (create after delete gets next seq, not the freed one) | `201` + DB assertions |
| Status free-switch all directions + history rows | `200` per transition + `STATUS_CHANGED` rows |
| Block unfinished → badge + `BLOCKED_SET`; block Done/archived → `409`; Done clears + `BLOCKED_CLEARED`; re-activate stays cleared | `200`/`409` + DB assertions |
| Assign → member only; cross-workspace/unknown assignee → `404 ASSIGNEE_NOT_MEMBER`; same-person reassign → no-op (no history, no event) | `200`/`404` + history-count assertions |
| Project attach → same-workspace active only; archived project → `409 PROJECT_ARCHIVED`; cross-workspace → `404` | `200`/`404`/`409` |
| Labels → create/rename conflict (case-insensitive) `409 LABEL_NAME_CONFLICT`; attach same-workspace only; double-attach `409`; detach missing `409`; delete unlinks (issues survive, count asserted) | `200`/`409` + join-count assertions |
| Archive → restore round trip preserves status + blocked | `200`, `archivedAt` set/cleared, `status`/`blocked` unchanged + `ARCHIVED`/`RESTORED` rows |
| Archive already archived / restore not archived | `409 ALREADY_ARCHIVED` / `409 NOT_ARCHIVED` |
| Update/attach/detach on archived issue | `409 ISSUE_ARCHIVED` |
| Delete — joins + history removed, sequence untouched, project/label rows survive | `200` + row-gone + name-reusable + seq-monotonic assertions |
| Delete rollback on cascade failure | issue still present |
| Project delete (F4 #8) → its issues `projectId` nulled + `PROJECT_CHANGED` rows, issues alive | cross-module test (calls projects route, asserts issues) |
| Member remove/leave → their issues unassigned (archived included) + `UNASSIGNED` rows | cross-module test (calls members route, asserts issues) |
| Archived workspace writes (#3–#7, #10–#14) | `409 WORKSPACE_ARCHIVED` |
| Archived workspace reads (#1,#2,#8,#9) | `200` |
| Cross-workspace — issue/label id from another workspace | `404 ISSUE_NOT_FOUND` / `404 LABEL_NOT_FOUND` scoped |
| List — each filter, AND-combination, labels-AND semantics, `q` text + `SHIP-###` exact, sort/order, cursor pagination (forward walk + invalid-cursor 400 + filter-change resets cursor) | returned sets + `nextCursor` assertions |
| History — chronological order, cursor walk, archived-issue readable | `200` + ordering assertions |
| View preference — `ISSUE` independent of `PROJECT`, absent ⇒ LIST | `200` + DB assertions |

### 10.2 Component tests (web) — MSW mocks of `/api/v1/workspaces/:slug/issues*` + `.../labels*`

| Surface | Cases |
|---|---|
| Issues page | Renders `issueCardSchema` list with labels inline; empty states (no issues / no filter matches / no labels); Kanban groups by status with empty columns visible; Archived view is list-only |
| View toggle + subviews | Segmented List/Kanban calls view-preference `PUT ISSUE`; persists; reload honors saved view; subview tabs map to §5.1 queries; list-only subviews never overwrite the saved preference |
| Issue card | Shows identifier, title, priority, assignee, project/cycle (cycle hidden in F5), due date, blocked badge |
| Permission-aware toolbar | Delete hidden for `MEMBER`; every other action visible to all roles |
| Create dialog/quick-add | Sends `POST #3`; assignee/project/label validation errors inline; success appears in Backlog |
| Edit/status/blocked controls | Send `PATCH #4`; `CANNOT_BLOCK_DONE` shows inline; Done-clears-blocked reflected immediately |
| Label manager | Sends `#10`–`#12`; `LABEL_NAME_CONFLICT` inline; delete shows `unlinkedIssues` count |
| Attach/detach | Sends `#13`/`#14`; double-attach/detach-missing states handled |
| Archive/restore/delete dialogs | Send `#5`/`#6` with `confirm:true`; delete requires typed `SHIP-###`, mismatch inline; archived card leaves board; restore from archived view returns it with status + blocked intact |
| Board drag | Drop issues `PATCH #4 {status}`; failure returns card + error; concurrent change refreshes latest |
| History panel | Renders `#8` chronological events; empty history state |
| Error envelope rendering | Every surface renders MSW-served `{error:{code,message}}` as friendly states, never raw dumps |
| Archived workspace/issue wrappers | All mutating affordances disabled with frozen/archived messaging |

Rules: components never re-implement business rules (e.g., "Member cannot delete" is API-enforced; web just hides controls). Tests assert wire behavior + rendered state.

### 10.3 End-to-end journey — golden path

Playwright against the composed stack (web + api + Postgres, reset between runs).

**Journey — issue lifecycle (core)**

```text
1. Owner + a Member exist in a workspace with a project and two labels (F3/F4)
2. Member creates "Fix login redirect" with priority + assignee (self) + project + label → SHIP-1 in Backlog
3. Member drags card Backlog→Todo→In Progress (or uses status control) → each flips + history grows
4. Member blocks it with a reason → badge appears; then moves it to Done → badge auto-clears
5. Owner reopens it to Todo → stays unblocked
6. Member attaches a second label, detaches the first → detail reflects both
7. Member archives it → gone from board; shown under Archived view
8. Member restores it → back with prior status
9. Member attempts delete → hidden/forbidden; Owner deletes with typed SHIP-1 → gone; next created issue is SHIP-2 (never reused)
```

**Negative E2E checks (cheap):**

- **Cross-workspace leak:** second workspace's issue id under first workspace's slug → 404.
- **Member delete attempt:** Member sends `DELETE #7` directly → 403; Owner succeeds.
- **Blocked Done:** attempt `PATCH {blocked:true}` on a Done issue → 409; move to Todo first, then block succeeds.
- **Archived frozen:** archive then attempt `PATCH`/attach → 409; restore then edit succeeds.
- **Archived workspace freeze:** archive the workspace, then attempt create/update/delete → 409; page still readable.

Scope discipline: journey + negatives are the mandatory F5 E2E suite; exhaustive cases stay in 10.1–10.2. Cycle/notification/comment/search/dashboard E2E lands with F6–F10.

---

## 11. Cross-cutting concerns

| Concern | Approach |
|---|---|
| **Rate limiting** | Per-route create (30/min per workspace), update (60/min), label ops (30/min), delete (10/min). Memory for MVP; global limiter exists; wiring finalized at F12. |
| **Validation encoding** | Due dates are `YYYY-MM-DD` strings end-to-end (`@db.Date`, data-model D10); api coerces to `Date`, web formats from the same schema. Colors are `#RRGGBB`; `q` bounded 2–200 chars. |
| **Sorting / ordering** | Server applies `sort`/`order` with `id` tiebreak; board groups by `status` client-side and applies the sort within columns (no manual ranking in MVP, spec §3.5). Server never stores order. Priority sort uses the D9 rank (Urgent first on `asc`). |
| **Pagination** | Cursor only (spec Q4 resolved). List (#1) cursors over the active `sort`; history (#8) over `(createdAt, id)`. Labels (#9) unpaginated (small set). A `LIMIT` cap is structural (`limit ≤ 100`), not a second pagination mode. |
| **Audit / history** | `issue_history` rows are the audit trail (data-model D7); written in-tx with every mutation. No separate audit table in F5. |
| **Progress derivation** | Project progress = `COUNT(status=DONE) / COUNT(*)` over non-archived issues per project (F4 card contract now live); cycle progress reuses the same query shape in F7. No stored counters. |
| **Search** | F5 `q` is bounded `ILIKE` + exact-identifier (this doc §5.1); F10 adds generated `tsvector` + GIN + grouped endpoint without changing these contracts. |
| **Notifications hook** | Issues exposes the assignment event inside its tx (§7/§8.1); F6 owns the `notification` rows. No notification rows in F5. |

---

## 12. References

- Shipyard: `features/issues/spec.md`, `features/issues/data-model.md`, `features/workspace/api-design.md` (guard chain, archived matrix, envelope), `features/members/api-design.md` (RBAC pipeline, `confirm: true`, unassign contract), `features/projects/api-design.md` (archive dimension, board-drag-as-PATCH, view-preference reuse, `SetNull` legs), `features/auth/api-design.md` (session, `emailVerified`), `features/notifications/spec.md` (F6 hook), `features/cycles/spec.md` (F7 relation), `features/comments/spec.md` (F8 cascade), `00-architecture.md` §5–§8, `ADR-001`–`ADR-003`, `Implementation Plan.md` F5
- Prisma: referential actions, compound indexes — `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- Cursor pagination: `https://www.prisma.io/docs/orm/prisma-client/queries/pagination#cursor-based-pagination`
- PostgreSQL `ILIKE` / pattern matching: `https://www.postgresql.org/docs/current/functions-matching.html`

---

*Next artifact: implementation (plan §5 Steps 3–7) — Prisma migration → module code (routes/controller/service/repository + shared schemas) → web slice (issues list/board, detail, labels manager, history panel, view toggle + subviews) → tests → `pnpm check`. Cycle relation + notification emission + comments cascade + full-text search land with F6–F8/F10 via the handoffs in §7.*
