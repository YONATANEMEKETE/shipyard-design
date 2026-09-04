# Cycles — Data Model

**Status:** Draft for review
**Last updated:** 2026-09-03
**Sources:** `features/cycles/spec.md` · `features/issues/data-model.md` (F5 precedent — archive dimension, functional unique index, `issue` planning legs, history pattern) · `features/projects/data-model.md` (F4 precedent — `archivedAt` dimension, name uniqueness, progress-derived-not-stored) · `features/members/data-model.md` (F3 precedent — partial indexes, cascade convention) · `features/workspace/data-model.md` (F2 precedent — identity, cascade) · `00-architecture.md` §5, §8, §9 · `ADR-001` (Prisma + Postgres) · `ADR-002` (shared contracts) · `Implementation Plan.md` F7
**Owner:** `apps/api` — Prisma-owned (hand-modeled, like workspace/members/projects/issues).

---

## 1. Overview

Cycles owns the **time-boxed iteration**: a named date range with a controlled lifecycle (Start / Complete / Reopen / Archive / Restore / Delete), a strict scheduling contract (no overlap among non-archived cycles, at most one Active per workspace), and derived progress through issues.

One new core table plus one column added to an existing table:

| Table / Change | Purpose | Formalized by |
|---|---|---|
| `cycle` | The iteration: identity, name, goal, date range, operational status, archive lifecycle | **F7 (this milestone)** |
| `issue.cycleId` | Nullable FK `issue → cycle` (`SetNull`); at most one cycle per issue | **F7 (migration on the F5 table)** |
| `IssueHistoryEvent` | Widened with `CYCLE_CHANGED` (additive, reuses F5 `issue_history`) | **F7** |

Progress is **derived, never stored** — `COUNT(status = DONE) / COUNT(*)` over non-archived issues per cycle (same rule as projects F4 and issues F5). Blocked never affects it (spec rule 11). Project↔cycle relationships are derived through shared issues and stored nowhere (rule 12).

`cycle` has no `ownerId` (unlike `project.ownerId`) — cycles are team-owned; all writes are gated by `WorkspaceRole` (Owner/Admin), not by a per-row owner. See D2.

---

## 2. Core schema (Prisma-owned)

### 2.1 `cycle`

One row per iteration. Immutable internal identity via `cuid()`; display name is never an identifier and is not directly unique — uniqueness is enforced by a functional index (D3, same pattern as `project_name_unique` / `label_name_unique`).

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK `@default(cuid())` | Immutable internal identifier. URL segment `/w/:slug/cycles/:id` (same reasoning as projects D5 — no cycle slug). |
| `workspaceId` | `String` | FK → `workspace.id` `onDelete: Cascade` + `@@index([workspaceId])` | Exactly one workspace (rule 1). Cascades with workspace (§7 cascade convention). |
| `name` | `String` | `@db.VarChar(120)` | Required, trimmed server-side. Unique per workspace via D3 index (archived reserve, delete releases — rule 2). |
| `goal` | `String?` | `@db.Text` | Optional goal/description (spec §2, PRD §5.7). Max 10,000 chars (matches projects/issues bound). |
| `status` | `CycleStatus` | `@default(PLANNED)` | Operational status `PLANNED \| ACTIVE \| COMPLETED`, changed **only** via controlled actions (D2). Creation lands in `PLANNED`. **No ARCHIVED value** — archive is `archivedAt` (D1). |
| `startDate` | `DateTime` | `@db.Date` | Required, day-precision (D4). Inclusive bound. Past dates allowed (spec Q2 resolved — cycles can start late). |
| `endDate` | `DateTime` | `@db.Date` | Required, day-precision (D4). Inclusive bound. Must be `>= startDate`. |
| `archivedAt` | `DateTime?` | `@@index([archivedAt])` | Presence ⇒ archived (read-only historical reference, excluded from scheduling). Kept on restore path; operational `status` untouched so restore is free (D1). |
| `createdAt` | `DateTime` | `@default(now())` | |
| `updatedAt` | `DateTime` | `@updatedAt` | |

> No soft-delete flag, no `deletedAt`: permanent delete (future Planned only) physically removes the row in one transaction (§6.6). Archive ≠ delete.
> No `progress` / `totalIssues` / `completedIssues` columns — derived from issues at read time (§6.7).
> No `ownerId` — team-owned (D2).

```prisma
enum CycleStatus {
  PLANNED
  ACTIVE
  COMPLETED
}

model Cycle {
  id          String      @id @default(cuid())
  workspaceId String
  name        String      @db.VarChar(120)
  goal        String?     @db.Text
  status      CycleStatus @default(PLANNED)
  startDate   DateTime    @db.Date
  endDate     DateTime    @db.Date
  archivedAt  DateTime?
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  issues    Issue[]

  @@index([workspaceId])
  @@index([status])
  @@index([archivedAt])
  @@index([workspaceId, status])
  @@map("cycle")
}
```

Uniqueness (raw SQL, D3 — same pattern as projects/labels):

```sql
CREATE UNIQUE INDEX cycle_name_unique
  ON cycle (workspace_id, lower(name));
```

Scheduling guards (raw SQL, D5/D6 — Prisma cannot express either):

```sql
-- D5: no overlap among non-archived cycles (inclusive dates).
-- Requires: CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE cycle ADD CONSTRAINT cycle_no_overlap
  EXCLUDE USING gist (
    workspace_id WITH =,
    daterange(start_date, end_date, '[]') WITH &&
  ) WHERE (archived_at IS NULL);

-- D6: at most one Active non-archived cycle per workspace.
CREATE UNIQUE INDEX cycle_single_active
  ON cycle (workspace_id) WHERE status = 'ACTIVE' AND archived_at IS NULL;
```

### 2.2 `issue.cycleId` — the F7 leg on the F5 table

Nullable, at most one cycle per issue (rule 1). Adding/removing is an **issue-level action** (spec §3.3) — the write path is the issues `PATCH` endpoint extended with `cycleId`, validated by the cycles service contract (§6.8).

| Column | Type | Attr | Notes |
|---|---|---|---|
| `cycleId` | `String?` | FK → `cycle.id` `onDelete: SetNull` + `@@index([cycleId])` + `@@index([workspaceId, cycleId])` | Nullable. Same-workspace invariant service-side (D7). Cycle delete clears via `SetNull` in the same tx (§6.6). Completing a cycle never touches it (rule 9). |

```prisma
model Issue {
  // ... existing F5 fields
  cycleId String?
  cycle   Cycle?  @relation(fields: [cycleId], references: [id], onDelete: SetNull)

  @@index([cycleId])
  @@index([workspaceId, cycleId])
}
```

`Cycle` gains the `issues Issue[]` back-relation above. F5's `CYCLE_CHANGED` history event is added to `IssueHistoryEvent` (additive widening, zero rewrite — same pattern as `ViewScope += ISSUE`):

```prisma
enum IssueHistoryEvent {
  // ... existing F5 values
  CYCLE_CHANGED // F7 adds
}
```

`Workspace` gains `cycles Cycle[]`.

### 2.3 No new preference scope

Cycles reuse the existing list/detail presentation without a List/Kanban toggle — no `ViewScope` widening in F7. Cycle issue views reuse the issues list endpoint with `?cycleId=` filter (Implementation Plan F7).

---

## 3. Key decisions & alternatives

### D1 — Archive is a separate dimension (`archivedAt`), not an enum value

**Decision:** identical to projects D1 and issues D1. `status` holds only the three operational values; archiving sets `archivedAt` (operational status untouched) and restore clears it. Board/list/scheduling exclusion is `WHERE archivedAt IS NULL`.

*Reason:* spec §3.2/§5-rule-8 requires restore to return to the stored pre-archive status. A single enum with `ARCHIVED` would force a second column to remember it. The dimension also gives the scheduling exclusion (`WHERE archived_at IS NULL` in D5) and the active-limit exclusion (D6) a single predicate to share.

*Lifecycle consequence:* only `PLANNED` / `COMPLETED` rows can be archived (Active must complete first — rule 6, enforced at service; the DB does not need a check constraint because the service is the only writer and the error shape must distinguish "complete first" from a constraint violation).

### D2 — Controlled transitions only; no generic status write; no per-row owner

**Decision:** `status` is never writable via a generic `PATCH { status }` (deliberate divergence from projects/issues board-drag). Every transition is a named action endpoint (Start / Complete / Reopen) with its own guards; archive/restore/delete are separate lifecycle exits. Writes require `OWNER | ADMIN` (spec §2 — view: all members); there is no `ownerId` column because accountability is team-level, not per-cycle.

*Considered and rejected:* reusing the projects free-switch `PATCH { status }` — would allow `PLANNED → COMPLETED` skips and `ACTIVE → PLANNED` regressions the spec forbids (rule 5). Named actions make illegal transitions unrepresentable at the route layer and let error codes name the guard that failed (`ANOTHER_ACTIVE_EXISTS`, `CYCLE_OVERLAP`, `COMPLETE_FIRST`).

### D3 — Case-insensitive workspace-scoped name uniqueness via raw functional index

**Decision:** `CREATE UNIQUE INDEX cycle_name_unique ON cycle (workspace_id, lower(name))`, names trimmed server-side. Archived rows reserve names; physical delete releases them (rule 2) — same semantics as projects D3 and labels D6.

*Rejected:* `@@unique([workspaceId, name])` (case-sensitive loophole); denormalized `nameKey` column (sync risk).

### D4 — Date-only fields use `@db.Date`; both required; inclusive bounds; past starts allowed

**Decision:** `startDate`/`endDate` are `DATE` (day-precision, no timezone-shift bugs — same as projects D4). Both required (spec §2). `endDate >= startDate` enforced at Zod + service (same-day cycles allowed: a one-day iteration is `start == end`, valid under inclusive semantics). Past `startDate` permitted (spec Q2); "future Planned" for delete is defined in §6.6 as `status = PLANNED AND startDate > today(server date)`.

*No length guardrails* (spec Q1 resolved): any valid non-overlapping range is accepted, from one day to multi-month.

### D5 — No-overlap enforced by a Postgres exclusion constraint (not application discipline)

**Decision:** `EXCLUDE USING gist (workspace_id WITH =, daterange(start_date, end_date, '[]') WITH &&) WHERE (archived_at IS NULL)` with `btree_gist` enabled. Checked at commit for create, date-edit, and restore — concurrent conflicting writes cannot both commit (one receives a constraint violation mapped to `409 CYCLE_OVERLAP`, §7 of `api-design.md`).

*Considered and rejected:* (a) service-only check (`SELECT overlapping …` then `INSERT`) — races under concurrency; two concurrent creates with the same range both read "no conflict" then both insert. Serializable transactions narrow but do not close this without predicate locking the whole workspace's ranges; the exclusion constraint is the standard Postgres answer. (b) Per-day expansion table — write amplification for zero benefit.

*Exclusion mechanics:* `daterange(..., '[]')` makes both bounds inclusive, matching spec §3.1 ("the next cycle starts after the previous ends" — `end = Jan 10`, `next start = Jan 11` OK; `next start = Jan 10` conflicts). Archived rows (`archived_at IS NOT NULL`) are excluded from the constraint — they neither block scheduling nor are blocked by it (spec §3.3).

### D6 — One-Active enforced by a partial unique index (not application discipline)

**Decision:** `CREATE UNIQUE INDEX cycle_single_active ON cycle (workspace_id) WHERE status = 'ACTIVE' AND archived_at IS NULL`. Start and Reopen race safely: two concurrent Starts — one commits, the other violates the index → `409 ANOTHER_ACTIVE_EXISTS` with the conflicting cycle's card in the error details (spec Q3 resolved).

*Why both D5 and D6:* overlap and active-limit are independent axes — two non-overlapping Planned cycles are legal (D5 passes, D6 untouched); two overlapping Planned cycles are illegal even with no Active (D5 fails); two non-overlapping cycles where both claim Active is illegal (D6 fails).

### D7 — `issue.cycleId` is `SetNull` with a same-workspace + non-archived-cycle invariant

**Decision:** FK `SetNull` completes both directions: cycle delete unassigns in the same tx (§6.6); issue delete needs no cycle-side cleanup. Service asserts on every attach: cycle exists in the same `workspaceId` and `cycle.archivedAt IS NULL` — assigning to an archived cycle is rejected (archived cycles are historical reference, spec §3.3). Detaching (`cycleId = null`) is always allowed, including from completed cycles (removing an issue from a Completed cycle is an issue-level correction, not a cycle mutation).

*Why the write path lives on issues:* spec §3.3 fixes adding/removing as issue-level actions, so the endpoint is issues `PATCH { cycleId }` (extended in F7), not a cycle-side `/cycles/:id/issues` writer. The cycles module owns the validation contract the issues service calls — same cross-module-service pattern as F3→F4 `transferOwnedProjects`.

### D8 — Progress derived, blocked excluded by construction

No counters stored. Progress = `total = COUNT(non-archived issues WHERE cycleId)`, `completed = COUNT(... AND status = DONE)`, `percent = completed / total (null when total = 0)`. `blocked` is never a predicate in the query, so rule 11 holds structurally rather than by discipline.

---

## 4. Shared contracts (`packages/shared`)

Added in F7, consumed by `api` and `web` (ADR-002). Mirrors the Prisma enums above.

```ts
// zod enums — mirror Prisma enums §2
export const cycleStatusSchema = z.enum(["PLANNED", "ACTIVE", "COMPLETED"]);

// canonical bounds — match DB column types
export const cycleNameSchema = z.string().trim().min(1).max(120);
export const cycleGoalSchema = z.string().max(10000).optional();
export const cycleDateSchema = z.string().regex(/^\d{4}-\d{2}-\d{2}$/); // YYYY-MM-DD

// request contracts owned by the cycles module
export const createCycleSchema = z.object({
  name: cycleNameSchema,
  goal: cycleGoalSchema,
  startDate: cycleDateSchema,   // required (spec §2)
  endDate: cycleDateSchema,     // required, >= startDate
});

export const updateCycleSchema = z.object({
  name: cycleNameSchema.optional(),
  goal: z.string().max(10000).nullable().optional(),
  startDate: cycleDateSchema.optional(),
  endDate: cycleDateSchema.optional(),
  // NOTE: no status field — transitions are named actions (D2), never generic writes.
});

// lifecycle actions carry no body beyond confirmation where required;
// guards (active-limit, overlap, complete-first) are evaluated server-side.
export const cycleLifecycleSchema = z.object({
  confirm: z.literal(true),
});

// F7 extension to the issues module contract (applied to updateIssueSchema):
// cycleId: z.string().cuid().nullable().optional()  // null ⇒ detach; omitted ⇒ leave as is

// response contracts
export const cycleProgressSchema = z.object({
  total: z.number().int().nonnegative(),
  completed: z.number().int().nonnegative(),
  percent: z.number().min(0).max(100).nullable(), // null when total === 0
});

export const cycleCardSchema = z.object({
  id: z.string(),
  workspaceId: z.string(),
  name: z.string(),
  status: cycleStatusSchema,
  startDate: cycleDateSchema,
  endDate: cycleDateSchema,
  archivedAt: z.string().datetime().nullable(),
  progress: cycleProgressSchema,   // derived per §6.7 (zeros when no issues)
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});

export const cycleDetailSchema = cycleCardSchema.extend({
  goal: z.string().nullable(),
});
```

`CYCLE_CHANGED` joins `issueHistoryEventSchema` (additive). Issue card/detail schemas gain `cycleId: z.string().nullable()` in F7 (read path; writes go through the extended `updateIssueSchema`).

---

## 5. Integrity invariants → spec rule mapping

| Spec rule | Enforcement point |
|---|---|
| 1 — every cycle in exactly one workspace; issue at most one cycle | `workspaceId` FK `Cascade` + index; `issue.cycleId` nullable singular FK |
| 2 — name unique per workspace, archived reserve, delete releases | D3 functional index `(workspace_id, lower(name))` over all non-deleted rows |
| 3 — non-archived ranges never overlap (inclusive); edits preserve | D5 exclusion constraint `cycle_no_overlap` (+ service pre-check for friendly errors) |
| 4 — only one Active per workspace | D6 partial unique index `cycle_single_active` (+ service pre-check) |
| 5 — lifecycle only via controlled actions | No generic status write (D2); named routes only (`api-design.md`) |
| 6 — Active cannot be archived | service precondition on `status` (archive rejects `ACTIVE` with `COMPLETE_FIRST`) |
| 7 — Completed read-only unless reopened; reopen respects guards | service read-only matrix + Start/Reopen guards re-evaluated in-tx |
| 8 — restore returns to stored status; must not conflict | `archivedAt` cleared, `status` untouched (D1); exclusion constraint re-evaluated at restore (conflict ⇒ `409 CYCLE_OVERLAP`) |
| 9 — completing changes no issue | no issue writes in the Complete transaction (§6.4) |
| 10 — deleting future Planned unassigns, never deletes issues | `issue.cycleId SetNull` in the same `$transaction` (§6.6) |
| 11 — blocked excluded from progress | progress query predicates on `status` only (D8) |
| 12 — no project↔cycle storage | no `projectId` on `cycle`; no `cycleId` on `project`; derivation through shared issues only |

Integrity summary — constraints added in F7:

| Constraint | Where | Purpose |
|---|---|---|
| `cycle_name_unique` functional unique | `cycle (workspace_id, lower(name))` | Case-insensitive name uniqueness per workspace |
| `cycle_no_overlap` exclusion | `cycle` (`btree_gist`, `WHERE archived_at IS NULL`) | Non-archived ranges never overlap (inclusive) |
| `cycle_single_active` partial unique | `cycle` (`WHERE status='ACTIVE' AND archived_at IS NULL`) | At most one Active per workspace |
| FK `cycle.workspaceId → workspace` `Cascade` | `cycle` | Cycle dies with its workspace |
| FK `issue.cycleId → cycle` `SetNull` | `issue` | Cycle delete unassigns; issue delete needs no cleanup |
| `@@index([workspaceId])` / `[status]` / `[archivedAt]` / `[workspaceId, status]` | `cycle` | List/active-lookup/filter hot paths |
| `@@index([cycleId])` / `[workspaceId, cycleId]` | `issue` | Cycle issue-list + progress hot paths |

---

## 6. Lifecycle semantics at the data layer

Every transition runs in a single Prisma `$transaction` with guards re-evaluated inside the transaction (read-then-write in one tx; the DB constraints D5/D6 are the backstop that makes concurrent transitions safe).

### 6.1 Creation (atomic, spec §4.1)

Single transaction inserts `cycle { workspaceId, name, goal, startDate, endDate, status: PLANNED }`. Name uniqueness (D3) and overlap (D5) are enforced at commit; the service pre-checks both first for friendly errors (`CYCLE_NAME_CONFLICT`, `CYCLE_OVERLAP` with the conflicting cycle's card). `endDate < startDate` rejected at Zod. No issues attached at creation (assignment is issue-level, §6.8).

### 6.2 Start: `PLANNED → ACTIVE` (spec §3.2)

Preconditions in-tx: `status = PLANNED`, `archivedAt IS NULL`, no other `ACTIVE` non-archived cycle in the workspace (D6 pre-check; violation ⇒ `ANOTHER_ACTIVE_EXISTS` + conflicting card), date range still non-overlapping (D5 pre-check; violation ⇒ `CYCLE_OVERLAP`). Then `UPDATE status = ACTIVE`. Issues untouched.

### 6.3 Complete: `ACTIVE → COMPLETED` (spec §3.2)

Preconditions: `status = ACTIVE`, `archivedAt IS NULL`. Then `UPDATE status = COMPLETED`. **No issue writes** (rule 9 — unfinished issues stay open in whatever status they hold). Cycle becomes read-only (§6.7 matrix) until reopened.

### 6.4 Reopen: `COMPLETED → ACTIVE` (spec §3.2)

Same guards as Start, re-evaluated at reopen time: no other Active (D6) and range still non-overlapping (D5) — a cycle created later may now occupy the range or the Active slot, in which case reopen fails with the same codes as Start (rule 7). Then `UPDATE status = ACTIVE`.

### 6.5 Archive / restore (reversible, spec §3.2)

- **Archive** (Owner/Admin, confirmed): preconditions `status ∈ {PLANNED, COMPLETED}` (`ACTIVE` ⇒ `409 COMPLETE_FIRST`, rule 6) and `archivedAt IS NULL`. `SET archivedAt = now()`; `status` **unchanged**. Archived cycles leave the scheduling constraint (D5 `WHERE` clause) and remain queryable as historical reference.
- **Restore** (confirmed): preconditions `archivedAt IS NOT NULL`. `SET archivedAt = NULL`; returns to stored `PLANNED`/`COMPLETED` automatically. The exclusion constraint re-evaluates — if a non-archived cycle now occupies the range, restore fails `409 CYCLE_OVERLAP` and the cycle stays archived (rule 8).

### 6.6 Permanent delete (irreversible, spec §3.2/rule 10)

Preconditions (service): `status = PLANNED`, `archivedAt IS NULL`, and `startDate > today(server date, day-precision)` — "future Planned … before its start date". Anything else ⇒ `409 CYCLE_NOT_DELETABLE` (covers Active, Completed, archived, and already-started Planned with a single code; the message names the reason).

Runs in **one `$transaction`**: `UPDATE issue SET cycleId = NULL WHERE cycleId = ?` (+ `CYCLE_CHANGED` history per affected issue, actor = deleter) **and** `DELETE FROM cycle WHERE id = ?`. If either fails, neither persists — issues are never deleted, only unassigned. The name is released for reuse (D3 over non-deleted rows only).

### 6.7 Read-only matrix + progress derivation

- `PLANNED` (non-archived): editable (name/goal/dates within D3/D5), startable, archivable, deletable-if-future.
- `ACTIVE` (non-archived): dates/name/goal frozen except via explicit edit rules (`api-design.md` — Active allows goal/name edits but date edits re-validate D5); completable; **not** archivable, **not** deletable.
- `COMPLETED` (non-archived): read-only except reopen/archive. Issue attach/detach to a Completed cycle is still allowed at the *issue* level (correction path, D7) — the cycle row itself is not mutated.
- Archived (`archivedAt` set, any status): fully read-only except restore/delete-eligibility (delete only if the stored status is `PLANNED` and the stored range is still future — evaluated against stored dates).
- Progress (read-time, D8): two `COUNT(*)` queries over `issue WHERE workspaceId AND cycleId AND archivedAt IS NULL`, grouped by `status = DONE` vs all. Returned inline on cycle cards/details; never persisted.

### 6.8 Issue ↔ cycle assignment (issue-level action, spec §3.3)

Implemented as the F7 extension of the issues `PATCH`: `updateIssueSchema += { cycleId: cuid.nullable.optional }`. Service (issues module, calling the cycles validation contract in-tx):

- `cycleId = <id>`: assert cycle in same workspace and `archivedAt IS NULL` (archived ⇒ `409 CYCLE_ARCHIVED`); assert issue not archived (`409 ISSUE_ARCHIVED`, F5 matrix); `SET cycleId` + `CYCLE_CHANGED` history `{old, new}`.
- `cycleId = null`: detach always allowed (even from Completed cycles); `CYCLE_CHANGED` history.
- Omitted: leave as is. Same-cycle set: no-op (no write, no history — mirrors F5 assignee/project no-op discipline).
- Assigning an issue to a cycle never changes the issue's `status`/`blocked`, and completing a cycle never changes its issues' rows (rules 9/11 both directions).

### 6.9 Archived-workspace interaction

`workspace.status = ARCHIVED` freezes the container (F2 guard): `GET` allowed, every cycle write rejected with `409 WORKSPACE_ARCHIVED` via `resolveWorkspaceContext({ rejectArchived: true })`. Service reasserts defensively. Archived-cycle read-only is the orthogonal per-row axis, enforced in the cycles service.

---

## 7. Forward handoffs — what this model does NOT contain

| Consumer | Contract F7 provides | Landed |
|---|---|---|
| **Issues (F5, already shipped)** | `issue.cycleId → cycle SetNull` + `CYCLE_CHANGED` event + `?cycleId=` list filter + cycle-issue progress counts. F5's `updateIssue`/`issueCard` shapes are extended, not replaced. | **F7 implements** — closes F5 §7 |
| **Notifications (F6)** | No cycle notifications in MVP (notifications spec §3.1: assignment + mentions only; explicitly no cycle-change notifications). No hook. | — (intentionally none) |
| **Comments (F8)** | No cycle-level comments; comments attach to issues only. Cycle detail links to issues; no new relation. | — (intentionally none) |
| **Dashboard (F9)** | Reads the current Active cycle (`WHERE status = ACTIVE AND archivedAt IS NULL`, at most one row per D6) + its progress + cycle issue lists. No new cycle columns; recently-viewed is F9's side effect. | **F9 implements** |
| **Search (F10)** | Generated `tsvector` on `cycle(name, goal)` + GIN; grouped search reads cycle cards. F7 ships plain reads only. | **F10 implements** |

---

## 8. Migration workflow

Hand-modeled Prisma (like workspace/members/projects/issues):

```bash
# 1 — add CycleStatus enum + Cycle model + Workspace.cycles back-relation
# 2 — add Issue.cycleId nullable FK (SetNull) + indexes
# 3 — widen IssueHistoryEvent with CYCLE_CHANGED
# 4 — run
pnpm --filter @shipyard/api db:migrate -- --name add_cycles_and_issue_cycle
pnpm --filter @shipyard/api db:generate
```

- The migration produces: 1 table (`cycle`), 1 enum (`CycleStatus`), 1 column (`issue.cycleId`) + FK + indexes, 1 widened enum (`IssueHistoryEvent`).
- Three raw-SQL appends in the same migration folder (Prisma cannot express them — same pattern as `project_name_unique`, `workspace_single_owner`, `invitation_single_pending`):
  1. `CREATE EXTENSION IF NOT EXISTS btree_gist;`
  2. `CREATE UNIQUE INDEX cycle_name_unique …` (D3)
  3. `ALTER TABLE cycle ADD CONSTRAINT cycle_no_overlap …` (D5) + `CREATE UNIQUE INDEX cycle_single_active …` (D6)
- Extension creation requires a role with `CREATE` on the database — Neon/local superuser covers this; the self-host compose notes it in `deployment.md` (F12). If the deploy role lacks extension rights, the migration fails fast with a clear `btree_gist` error rather than silently shipping without the overlap guard.
- The F1 Testcontainers harness applies migrations automatically each test run.

**Post-migration verification (manual, once):**

```sql
-- no duplicate case-insensitive names per workspace
SELECT workspace_id, lower(name), count(*) FROM cycle GROUP BY 1,2 HAVING count(*)>1;
-- no overlapping non-archived ranges per workspace (should return zero rows)
SELECT a.workspace_id, a.id, b.id FROM cycle a JOIN cycle b
  ON a.workspace_id = b.workspace_id AND a.id < b.id
  AND a.archived_at IS NULL AND b.archived_at IS NULL
  AND daterange(a.start_date, a.end_date, '[]') && daterange(b.start_date, b.end_date, '[]');
-- at most one active non-archived cycle per workspace
SELECT workspace_id, count(*) FILTER (WHERE status='ACTIVE' AND archived_at IS NULL)
  FROM cycle GROUP BY workspace_id HAVING count(*) FILTER (WHERE status='ACTIVE' AND archived_at IS NULL) > 1;
```

---

## 9. What we intentionally do NOT model

| Deferred | Why |
|---|---|
| `cycle.ownerId` / per-cycle ownership | Team-owned by design (D2); RBAC comes from `WorkspaceRole`, not a row owner. |
| Generic `PATCH { status }` on cycles | Forbidden by D2 — transitions are named actions with distinct guards. |
| Recurring cycles / auto-carryover of unfinished issues | Spec §6 out of scope. |
| Velocity, burndown, capacity, workload fields | Spec §6 out of scope; progress is two counts (D8). |
| `tsvector` / full-text indexes on cycles | Owned by Search (F10); F7 ships plain reads. |
| Cycle-level comments / notifications | Explicitly out of MVP (notifications spec §3.1; comments attach to issues). |
| Soft delete / trash | Archive covers reversible hiding; delete is confirmed and physical (§6.6). |
| Cycle length guardrails | None per spec Q1 — any valid non-overlapping range (D4). |

---

## 10. Open product questions — resolved at data layer

| Spec §7 | Decision |
|---|---|
| 1 — cycle length guardrails | **None.** Any `endDate >= startDate` non-overlapping range accepted (D4). |
| 2 — start date in the past | **Allowed.** Past `startDate` valid; "future" for delete means `startDate > today` (§6.6). |
| 3 — start conflict UX | **Server returns the conflicting cycle's card** in `ANOTHER_ACTIVE_EXISTS` / `CYCLE_OVERLAP` error details so the UI can name it ("complete *Sprint 12* first") — specified in `api-design.md` §7. |

---

## 11. References

- Shipyard: `features/cycles/spec.md`, `features/issues/data-model.md` (archive dimension D1, `issue` legs, history pattern), `features/projects/data-model.md` (functional index, derived progress), `features/members/data-model.md` (partial index pattern, cascade convention), `features/workspace/data-model.md` (identity, cascade), `features/notifications/spec.md` (no cycle notifications), `00-architecture.md` §5/§8/§9, `ADR-001`, `ADR-002`, `Implementation Plan.md` F7
- Prisma indexes & referential actions: `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- PostgreSQL exclusion constraints: `https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-EXCLUSION`
- PostgreSQL `btree_gist` + range overlap: `https://www.postgresql.org/docs/current/btree-gist.html` + `https://www.postgresql.org/docs/current/rangetypes.html`
- PostgreSQL partial unique indexes: `https://www.postgresql.org/docs/current/indexes-partial.html`

---

*Next artifact: `api-design.md` — endpoint inventory over `cycle` + `issue.cycleId` extension (CRUD, Start/Complete/Reopen/Archive/Restore/Delete, cycle issue-list/progress reads), guard chain per route (manage Owner/Admin, view any member), error codes with conflicting-cycle details, and the issues-PATCH wiring.*
