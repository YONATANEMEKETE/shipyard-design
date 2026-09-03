# Issues — Data Model

**Status:** Draft for review
**Last updated:** 2026-09-03
**Sources:** `features/issues/spec.md` · `features/projects/data-model.md` (F4 precedent — archive dimension, functional unique index, `transferOwnedProjects` contract, view_preference) · `features/members/data-model.md` (F3 precedent — partial indexes, cascade convention, guard chain) · `features/workspace/data-model.md` (F2 precedent — identity, cascade) · `features/auth/data-model.md` (identity — `user` table) · `00-architecture.md` §5, §8, §9 · `ADR-001` (Prisma + Postgres + Better Auth) · `ADR-002` (shared contracts) · `Implementation Plan.md` F5
**Owner:** `apps/api` — Prisma-owned (hand-modeled, like workspace/members/projects).

---

## 1. Overview

Issues owns the **unit of work**: creation with a stable display identifier, fixed workflow statuses, priority, assignment, planning (project now, cycle in F7), labels, due dates, blocked state, archive lifecycle, and per-issue history.

Five tables, one widened enum:

| Table | Purpose | Formalized by |
|---|---|---|
| `issue` | The work item: identity, title, status, priority, assignment, planning, blocked, archive | **F5 (this milestone)** |
| `label` | Workspace-scoped tag: name + color | **F5** |
| `issue_label` | Join: many-to-many issue ↔ label | **F5** |
| `issue_history` | Append-only per-issue audit: status/blocked/assignee/planning changes | **F5** |
| `workspace_issue_sequence` | Per-workspace counter backing `SHIP-###` identifiers | **F5** |
| `ViewScope` | Widened with `ISSUE` (additive, reuses F4 `view_preference`) | **F5** |

`project` is owned by F4 — F5 wires the reciprocal leg (`issue.projectId → project`, `onDelete: SetNull`) that F4's delete transaction already anticipates (projects `data-model.md` §6.4/§7). `cycle` is **not** modeled here — per `Implementation Plan.md` F5, the Cycle relation lands in F7 with its own migration; F5 contracts omit `cycleId` entirely (§7). Notifications (F6) and Comments (F8) are consumers, not tables here — F5 exposes the assignment-event contract they implement (§7).

`issue.assigneeId` and `issue.creatorId` reference `user.id` (not membership row ids) because assignment and history operate on user ids and membership is a service invariant — see D3.

---

## 2. Core schema (Prisma-owned)

### 2.1 `issue`

One row per work item. Immutable internal `id` (`cuid()`); human identifier is `SHIP-{seqNumber}` derived from the per-workspace sequence (D2). Display name is never an identifier.

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK `@default(cuid())` | Immutable internal identifier. URL segment `/w/:slug/issues/:id`. |
| `workspaceId` | `String` | FK → `workspace.id` `onDelete: Cascade` + `@@index([workspaceId])` | Exactly one workspace (spec rule 1). Cascades with workspace (§7 cascade convention). |
| `seqNumber` | `Int` | `@@unique([workspaceId, seqNumber])` | Per-workspace sequence value. Displayed as `SHIP-{seqNumber}` (D2). Never reused, never decremented. |
| `title` | `String` | `@db.VarChar(255)` | Required, trimmed server-side. Only mandatory field (spec §3.1). |
| `description` | `String?` | `@db.Text` | Optional, max 10,000 chars (matches projects bound). |
| `status` | `IssueStatus` | `@default(BACKLOG)` | Fixed workflow `BACKLOG \| TODO \| IN_PROGRESS \| DONE`, free transitions any direction (spec §3.2). No ARCHIVED value — archive is `archivedAt` (D1). |
| `priority` | `IssuePriority` | `@default(NO_PRIORITY)` | `NO_PRIORITY \| URGENT \| HIGH \| MEDIUM \| LOW` (spec Q1 resolved). Sort rank Urgent > High > Medium > Low > No Priority (D9). |
| `assigneeId` | `String?` | FK → `user.id` `onDelete: SetNull` + `@@index([assigneeId])` | Nullable. Must be a current workspace member when set (service invariant, D3). Unset on member remove/leave (§6.6). |
| `creatorId` | `String` | FK → `user.id` `onDelete: Restrict` + `@@index([creatorId])` | Required. Every issue has one creator (spec rule 3). `Restrict` — user delete blocked until handled; no user-deletion path in MVP (D3). |
| `projectId` | `String?` | FK → `project.id` `onDelete: SetNull` + `@@index([projectId])` | Nullable, at most one project (rule 2). Same-workspace + non-archived invariant service-side (D4). Project delete clears via `SetNull` in same tx (F4 §6.4). |
| `dueDate` | `DateTime?` | `@db.Date` | Optional, day-precision like projects D4. Past dates allowed (overdue is a feature). |
| `blocked` | `Boolean` | `@default(false)` | Orthogonal flag, not a status (spec §3.3, D9). |
| `blockedReason` | `String?` | `@db.VarChar(500)` | Optional, only when `blocked = true`. Cleared atomically with flag (D9). Empty string normalized to `null`. |
| `archivedAt` | `DateTime?` | `@@index([archivedAt])` | Presence ⇒ archived (read-only, hidden from active lists/boards). Kept on restore path; `status` + `blocked` untouched so restore is free (D1). |
| `createdAt` | `DateTime` | `@default(now())` | |
| `updatedAt` | `DateTime` | `@updatedAt` | |

> No `cycleId` column in F5 — added with FK in F7 (§7). F5 request contracts omit it.
> No `progress` or denormalized counts — project/cycle progress is derived from issue `status` queries in F5/F7, never stored.
> No soft-delete flag, no `deletedAt` — delete physically removes the row + descendants in one transaction (§6.5). Archive ≠ delete.
> No `tsvector` column in F5 — F5 search is `ILIKE title/description + exact SHIP-###`; generated `tsvector` + GIN lands in F10 Search (§7).

```prisma
enum IssueStatus {
  BACKLOG
  TODO
  IN_PROGRESS
  DONE
}

enum IssuePriority {
  NO_PRIORITY
  URGENT
  HIGH
  MEDIUM
  LOW
}

model Issue {
  id            String        @id @default(cuid())
  workspaceId   String
  seqNumber     Int
  title         String        @db.VarChar(255)
  description   String?       @db.Text
  status        IssueStatus   @default(BACKLOG)
  priority      IssuePriority @default(NO_PRIORITY)
  assigneeId    String?
  creatorId     String
  projectId     String?
  dueDate       DateTime?     @db.Date
  blocked       Boolean       @default(false)
  blockedReason String?       @db.VarChar(500)
  archivedAt    DateTime?
  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  assignee  User?     @relation("AssignedIssues", fields: [assigneeId], references: [id], onDelete: SetNull)
  creator   User      @relation("CreatedIssues", fields: [creatorId], references: [id], onDelete: Restrict)
  project   Project?  @relation(fields: [projectId], references: [id], onDelete: SetNull)

  labels  IssueLabel[]
  history IssueHistory[]

  @@unique([workspaceId, seqNumber])
  @@index([workspaceId])
  @@index([status])
  @@index([priority])
  @@index([assigneeId])
  @@index([creatorId])
  @@index([projectId])
  @@index([dueDate])
  @@index([blocked])
  @@index([archivedAt])
  @@index([workspaceId, status])
  @@index([workspaceId, assigneeId])
  @@index([workspaceId, projectId])
  @@map("issue")
}
```

`Workspace`, `User`, `Project` gain back-relations (`issues`, `assignedIssues`, `createdIssues`, `projectIssues`).

### 2.2 `label` — workspace-scoped tag (spec §3.7)

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK `@default(cuid())` | |
| `workspaceId` | `String` | FK → `workspace.id` `onDelete: Cascade` + `@@index([workspaceId])` | Scope. Cascades with workspace. |
| `name` | `String` | `@db.VarChar(60)` | Required, trimmed. Unique per workspace via D6 functional index. |
| `color` | `String` | `@db.VarChar(7)` | Hex `#RRGGBB`, validated. Default `#6B7280` (gray). Any member may recolor (spec Q2 resolved). |
| `createdAt` | `DateTime` | `@default(now())` | |
| `updatedAt` | `DateTime` | `@updatedAt` | |

```prisma
model Label {
  id          String   @id @default(cuid())
  workspaceId String
  name        String   @db.VarChar(60)
  color       String   @db.VarChar(7)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  workspace Workspace    @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  issues    IssueLabel[]

  // D6: case-insensitive workspace-scoped uniqueness is a raw SQL
  // functional index (label_name_unique) appended to the migration.
  @@index([workspaceId])
  @@map("label")
}
```

Uniqueness (raw SQL, D6 — same pattern as `project_name_unique`):

```sql
CREATE UNIQUE INDEX label_name_unique
  ON label (workspace_id, lower(name));
```

### 2.3 `issue_label` — join (many-to-many)

Descendant child — cascades from both sides; no `workspaceId` denormalization needed because workspace delete cascades `workspace → issue → join` and `workspace → label → join` (both paths converge; either suffices for full cleanup).

| Column | Type | Attr | Notes |
|---|---|---|---|
| `issueId` | `String` | FK → `issue.id` `onDelete: Cascade` | |
| `labelId` | `String` | FK → `label.id` `onDelete: Cascade` | Service asserts same `workspaceId` on attach (D6). |
| `createdAt` | `DateTime` | `@default(now())` | |

```prisma
model IssueLabel {
  issueId   String
  labelId   String
  createdAt DateTime @default(now())

  issue Issue @relation(fields: [issueId], references: [id], onDelete: Cascade)
  label Label @relation(fields: [labelId], references: [id], onDelete: Cascade)

  @@id([issueId, labelId])
  @@index([labelId])
  @@map("issue_label")
}
```

Deleting a label unlinks it (`DELETE label` cascades joins, issues untouched — spec §3.7). Removing an issue from a project/cycle never touches labels.

### 2.4 `issue_history` — append-only audit (spec rule 11)

Records status, blocked, assignee, and planning changes. Not a global activity feed — Dashboard (F9) derives its feed by reading this table alongside others; no separate activity table in F5.

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK `@default(cuid())` | |
| `workspaceId` | `String` | FK → `workspace.id` `onDelete: Cascade` + `@@index([workspaceId])` | Denormalized scope: workspace delete cleans history even if issue row is already gone mid-tx; also powers workspace activity queries without joining `issue`. |
| `issueId` | `String` | FK → `issue.id` `onDelete: Cascade` + `@@index([issueId])` | Issue delete removes its history (spec rule 9 — history dies with issue). |
| `actorId` | `String?` | FK → `user.id` `onDelete: SetNull` + `@@index([actorId])` | Who made the change. `SetNull` preserves history if the actor is later deleted. Nullable for that reason. |
| `event` | `IssueHistoryEvent` | — | Typed event (see enum). One row per atomic change; multi-field updates emit one row per changed concern. |
| `oldValue` | `String?` | `@db.Text` | Prior value serialized as plain text (status/priority/userId/projectId/ISO date/blocked flag). `null` = previously unset. |
| `newValue` | `String?` | `@db.Text` | New value, same encoding. |
| `createdAt` | `DateTime` | `@default(now())` + `@@index([issueId, createdAt])` | Chronological order. No `updatedAt` — rows are immutable. |

```prisma
enum IssueHistoryEvent {
  CREATED
  STATUS_CHANGED
  BLOCKED_SET
  BLOCKED_CLEARED
  ASSIGNED
  UNASSIGNED
  PRIORITY_CHANGED
  PROJECT_CHANGED
  DUE_DATE_CHANGED
  TITLE_CHANGED
  ARCHIVED
  RESTORED
  LABEL_ADDED
  LABEL_REMOVED
  // F7 adds: CYCLE_CHANGED (additive widening, zero rewrite)
}

model IssueHistory {
  id          String            @id @default(cuid())
  workspaceId String
  issueId     String
  actorId     String?
  event       IssueHistoryEvent
  oldValue    String?           @db.Text
  newValue    String?           @db.Text
  createdAt   DateTime          @default(now())

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  issue     Issue     @relation(fields: [issueId], references: [id], onDelete: Cascade)
  actor     User?     @relation(fields: [actorId], references: [id], onDelete: SetNull)

  @@index([workspaceId])
  @@index([issueId])
  @@index([actorId])
  @@index([issueId, createdAt])
  @@map("issue_history")
}
```

History is write-only from the API (no update/delete endpoints); reads are chronological per issue. Label attach/detach emits `LABEL_ADDED`/`LABEL_REMOVED` with the label id as value. Description edits do **not** emit history rows in MVP (bounded noise; description is visible on the issue itself).

### 2.5 `workspace_issue_sequence` — identifier counter (D2)

| Column | Type | Attr | Notes |
|---|---|---|---|
| `workspaceId` | `String` | PK, FK → `workspace.id` `onDelete: Cascade` | One row per workspace. Created lazily on first issue create (or eagerly on workspace create — implementation choice, same semantics). |
| `nextNumber` | `Int` | `@default(1)` | Next `seqNumber` to allocate. Only ever increments, never decrements — deletion never reuses (spec rule 4). |

```prisma
model WorkspaceIssueSequence {
  workspaceId String @id
  nextNumber  Int    @default(1)

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@map("workspace_issue_sequence")
}
```

Allocation is `UPDATE workspace_issue_sequence SET nextNumber = nextNumber + 1 WHERE workspaceId = ? RETURNING nextNumber - 1` (or Prisma `upsert` + `increment` in the same `$transaction` as the issue insert) — atomic under concurrency; two concurrent creates never receive the same number.

`Workspace` gains `issues`, `labels`, `issueHistories`, `issueSequence` back-relations. `User` gains `assignedIssues`, `createdIssues`, `issueHistoryActions`.

### 2.6 `ViewScope` widening (F4 → F5)

```prisma
enum ViewScope {
  PROJECT
  ISSUE // F5 adds — additive, existing PROJECT rows untouched
}
```

No new preference table. `view_preference` rows with `scope = ISSUE` are read/written by the same helpers with rule-12 independence (toggling issues never touches a `PROJECT` row).

---

## 3. Key decisions & alternatives

### D1 — Archive is a separate dimension (`archivedAt`), status + blocked preserved

**Decision:** identical to projects D1. `status` holds only the four workflow values; `blocked`/`blockedReason` stay untouched by archive/restore. Archive = `SET archivedAt = now()`; restore = `SET archivedAt = NULL`. Board/list hiding is `WHERE archivedAt IS NULL`; Archived view is `WHERE archivedAt IS NOT NULL`.

*Reason:* spec §3.6 requires restore to return to prior status **and** blocked state. A status-enum `ARCHIVED` would force a second column to remember both. The dimension preserves both for free and keeps `ARCHIVED` out of the Kanban columns (spec §3.5: archived never on board).

### D2 — Display identifier via per-workspace sequence table, constant `SHIP-` prefix

**Decision:** `SHIP-{seqNumber}` where `seqNumber` comes from `workspace_issue_sequence.nextNumber` (atomic increment in the create transaction) with `@@unique([workspaceId, seqNumber])` as the backstop. Prefix is the constant `SHIP-` for MVP (spec Q3 resolved); no per-workspace prefix derivation, no `workspace` schema change.

*Considered and rejected:* (a) `count(*) + 1` — races under concurrency and reuses numbers after delete, violating rule 4; (b) global DB sequence — not workspace-scoped, leaks cross-workspace ordering; (c) per-workspace native sequences (`CREATE SEQUENCE per workspace`) — DDL per workspace, unmanageable in migrations and self-host; (d) workspace-derived prefix (e.g. initials of name) — requires immutable prefix column on `workspace`, collision handling on rename, and display-parsing complexity for zero MVP value. Revisit post-MVP if human prefixes are demanded.

Display format: `SHIP-24` (no zero-padding — padding is presentational; the API returns `seqNumber: Int` + `identifier: "SHIP-24"` and the web renders `identifier` verbatim).

### D3 — `assigneeId` → `user.id` `SetNull` (nullable); `creatorId` → `user.id` `Restrict` (required)

**Decision:** assignment references the user, not the membership row (membership ids are workspace-scoped handles for role ops per F3; assignment is about the human). "Assignee is a current member" is a service invariant checked on every set (resolve `workspace_member` for `(workspaceId, assigneeUserId)` or reject).

`SetNull` vs `Restrict` split is deliberate: unassigned is a legal issue state, so member removal unassigns (`SET assigneeId = NULL`, §6.6) and a hypothetical user delete must not block on assigned issues — hence `SetNull`. Creator is mandatory per rule 3 and has no "uncreated" state, so `Restrict` forces correct sequencing (transfer/handle first), mirroring `project.ownerId` D2.

### D4 — `projectId` → `project.id` `SetNull`; same-workspace + non-archived invariant

**Decision:** FK `SetNull` completes the F4 §6.4 contract: project delete clears assignments in the same transaction (issues survive). Service asserts on every attach: project exists in the same `workspaceId` and `archivedAt IS NULL` — assigning to an archived project is rejected (archived projects are read-only per F4). Detaching (`projectId = null`) is always allowed.

### D5 — No `cycleId` column in F5

**Decision:** omit entirely; F7 adds `cycleId String? → cycle.id SetNull` with its own migration plus validation integration into issue create/update. Shipping an unvalidated `cycleId String?` now would accept values that mean nothing and fork the contract; omitting keeps F5 honest per `Implementation Plan.md` F5 ("should not create a fake cycle implementation").

### D6 — Labels: functional unique index + join-table unlink semantics

**Decision:** `CREATE UNIQUE INDEX label_name_unique ON label (workspace_id, lower(name))` (raw SQL, same pattern as `project_name_unique`); names trimmed server-side. Any member may create/rename/recolor/delete labels (spec Q2 resolved — labels are low-risk shared vocabulary; gating to Owner/Admin would bottleneck the core workflow). Attach asserts label and issue share `workspaceId`. Label delete cascades joins only.

*Considered and rejected:* `@@unique([workspaceId, name])` — case-sensitive, allows `bug` + `Bug`; denormalized `nameKey` column — sync risk, same rejection as projects D3.

### D7 — History is a typed append-only table, not a generic JSON log

**Decision:** `IssueHistoryEvent` enum + `oldValue`/`newValue` text pair. Typed events keep the "what changed" queryable (`WHERE event = STATUS_CHANGED`) without parsing JSON, and the text pair is sufficient for every MVP field (ids, enums, ISO dates, `true/false`). No `updatedAt`, no update/delete paths — rows are immutable facts.

*Considered and rejected:* (a) single JSON `diff` column — unqueryable without Postgres JSON operators and harder to render per-event in the UI; (b) reusing a future global `activity` table now — Dashboard (F9) needs cross-entity aggregation; per-entity history tables composed at read time keep module ownership clean (each module owns its facts).

### D8 — `ViewScope` widened, not duplicated (F4 D6 implementation)

Additive `ISSUE` value, zero data rewrite. Helpers `getViewPreference`/`setViewPreference` take `scope`; absence ⇒ `LIST`.

### D9 — Blocked orthogonal; priority rank fixed; status transitions free

`blocked` is an independent boolean; only `BACKLOG | TODO | IN_PROGRESS` may set it (service rejects on `DONE` or archived). `DONE` transition clears `blocked + blockedReason` atomically and emits `BLOCKED_CLEARED` + `STATUS_CHANGED`. Re-activating never restores (spec §3.3/rule 6). Status changes accept any direction with no confirmation; every change emits history. Priority rank for sorting: `URGENT(0) < HIGH(1) < MEDIUM(2) < LOW(3) < NO_PRIORITY(4)`.

### D10 — Day-precision dates (`@db.Date`)

Same as projects D4: `dueDate` is a `DATE`; API/web exchange `YYYY-MM-DD` strings; past dates permitted (powers overdue views).

### D11 — Title/description/reason bounds

`title VarChar(255)` (matches `issue` ergonomics; longer belongs in description), `description Text ≤10k` (matches projects), `blockedReason VarChar(500)`, label `name VarChar(60)`, comment bound deferred to F8. Bounds enforced in Zod (shared) and matched by DB column types.

---

## 4. Shared contracts (`packages/shared`)

Added in F5, consumed by `api` and `web` (ADR-002). Mirrors the Prisma enums above.

```ts
// zod enums — mirror Prisma enums §2
export const issueStatusSchema = z.enum(["BACKLOG", "TODO", "IN_PROGRESS", "DONE"]);
export const issuePrioritySchema = z.enum(["NO_PRIORITY", "URGENT", "HIGH", "MEDIUM", "LOW"]);
export const issueHistoryEventSchema = z.enum([
  "CREATED", "STATUS_CHANGED", "BLOCKED_SET", "BLOCKED_CLEARED",
  "ASSIGNED", "UNASSIGNED", "PRIORITY_CHANGED", "PROJECT_CHANGED",
  "DUE_DATE_CHANGED", "TITLE_CHANGED", "ARCHIVED", "RESTORED",
  "LABEL_ADDED", "LABEL_REMOVED",
  // F7 adds: "CYCLE_CHANGED"
]);

// view scope widened (F4 D6): PROJECT | ISSUE
export const viewScopeSchema = z.enum(["PROJECT", "ISSUE"]);

// canonical bounds — match DB column types (D11)
export const issueTitleSchema = z.string().trim().min(1).max(255);
export const issueDescriptionSchema = z.string().max(10000).optional();
export const blockedReasonSchema = z.string().trim().max(500).optional();
export const labelNameSchema = z.string().trim().min(1).max(60);
export const labelColorSchema = z.string().regex(/^#[0-9A-Fa-f]{6}$/);
export const issueDateSchema = z.string().regex(/^\d{4}-\d{2}-\d{2}$/); // YYYY-MM-DD

// request contracts owned by the issues module
export const createIssueSchema = z.object({
  title: issueTitleSchema,
  description: issueDescriptionSchema,
  priority: issuePrioritySchema.optional(),       // omitted ⇒ NO_PRIORITY
  status: issueStatusSchema.optional(),          // omitted ⇒ BACKLOG (allows create-into-column)
  assigneeId: z.string().cuid().nullable().optional(), // user id, same workspace
  projectId: z.string().cuid().nullable().optional(),
  labelIds: z.array(z.string().cuid()).max(20).optional(),
  dueDate: issueDateSchema.nullable().optional(),
  // NOTE: no cycleId in F5 — added by F7 (D5).
});

export const updateIssueSchema = z.object({
  title: issueTitleSchema.optional(),
  description: z.string().max(10000).nullable().optional(),
  status: issueStatusSchema.optional(),
  priority: issuePrioritySchema.optional(),
  assigneeId: z.string().cuid().nullable().optional(), // null ⇒ unassign
  projectId: z.string().cuid().nullable().optional(),  // null ⇒ detach
  dueDate: issueDateSchema.nullable().optional(),
  blocked: z.boolean().optional(),
  blockedReason: blockedReasonSchema.nullable().optional(),
  // label membership is managed via dedicated attach/detach endpoints, not here.
});

export const createLabelSchema = z.object({
  name: labelNameSchema,
  color: labelColorSchema.optional(), // omitted ⇒ #6B7280
});

export const updateLabelSchema = z.object({
  name: labelNameSchema.optional(),
  color: labelColorSchema.optional(),
});

// response contracts
export const issueAssigneeCardSchema = z.object({
  userId: z.string(),
  name: z.string(),
  email: z.string().email(),
  image: z.string().nullable(),
});

export const labelCardSchema = z.object({
  id: z.string(),
  workspaceId: z.string(),
  name: z.string(),
  color: z.string(),
});

export const issueCardSchema = z.object({
  id: z.string(),
  workspaceId: z.string(),
  seqNumber: z.number().int().positive(),
  identifier: z.string(), // SHIP-{seqNumber}, rendered verbatim
  title: z.string(),
  status: issueStatusSchema,
  priority: issuePrioritySchema,
  assignee: issueAssigneeCardSchema.nullable(),
  projectId: z.string().nullable(),
  // F7 adds: cycleId
  dueDate: issueDateSchema.nullable(),
  blocked: z.boolean(),
  blockedReason: z.string().nullable(),
  labels: z.array(labelCardSchema),
  archivedAt: z.string().datetime().nullable(),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});

export const issueDetailSchema = issueCardSchema.extend({
  description: z.string().nullable(),
  creator: issueAssigneeCardSchema,
});

export const issueHistoryCardSchema = z.object({
  id: z.string(),
  event: issueHistoryEventSchema,
  actor: issueAssigneeCardSchema.nullable(),
  oldValue: z.string().nullable(),
  newValue: z.string().nullable(),
  createdAt: z.string().datetime(),
});
```

List/query/filter/sort/pagination contracts live in `api-design.md` (cursor pagination per spec Q4 resolved) and reuse these card schemas — no parallel shapes.

---

## 5. Integrity invariants → spec rule mapping

| Spec rule | Enforcement point |
|---|---|
| 1 — every issue in exactly one workspace; planning scope workspace-local | `workspaceId` FK `Cascade` + index; assignee/project/labels validated same-workspace at service |
| 2 — at most one project (and in F7, one cycle); project↔cycle derived | `projectId` nullable singular FK; no `project↔cycle` column anywhere |
| 3 — one creator, one status, one priority | `creatorId` non-null `Restrict`; `status`/`priority` non-null enums with defaults |
| 4 — identifiers unique per workspace, never reused | `@@unique([workspaceId, seqNumber])` + monotonic `workspace_issue_sequence` (D2) |
| 5 — defaults Backlog / No Priority / unblocked | DB defaults + service defaults on omitted fields |
| 6 — only unfinished blockable; Done clears; re-activate never restores | service preconditions on `status` + atomic clear on `→ DONE` (D9) |
| 7 — blocked never affects progress | no progress column; progress derived from `status` only in queries |
| 8 — archived read-only; restore preserves status + blocked | `archivedAt` dimension (D1); service read-only matrix |
| 9 — delete permanent; descendants die; identifier never reused | `issue` delete cascades `issue_label` + `issue_history` (+ F8 comments, F6 notifications per §7); sequence never decrements |
| 10 — Members create/edit/archive/restore; Owner/Admin delete only | guard chain: writes any member, delete `OWNER\|ADMIN` (`api-design.md`) |
| 11 — history records status/blocked/assignee/planning | `issue_history` writes inside the same tx as every mutation (§6) |
| 12 — view pref per user per workspace, independent, list-only never overwrites | reused `view_preference`, `@@unique([workspaceId, userId, scope])`, absent ⇒ LIST |
| 13 — assignee change notifies; same-person no-op | assignment-event contract for F6 (§7); service detects actual change before emitting |

Integrity summary — constraints added in F5:

| Constraint | Where | Purpose |
|---|---|---|
| `@@unique([workspaceId, seqNumber])` | `issue` | Identifier uniqueness per workspace |
| `label_name_unique` functional unique | `label (workspace_id, lower(name))` | Case-insensitive label uniqueness per workspace |
| `@@id([issueId, labelId])` | `issue_label` | No duplicate label on one issue |
| FK `issue.workspaceId → workspace` `Cascade` | `issue` | Issue dies with workspace |
| FK `issue.projectId → project` `SetNull` | `issue` | Project delete unassigns (F4 leg) |
| FK `issue.assigneeId → user` `SetNull` | `issue` | Member remove/leave unassigns; user delete safe |
| FK `issue.creatorId → user` `Restrict` | `issue` | Creator mandatory, never orphaned |
| FK cascades `issue → join/history` | `issue_label`, `issue_history` | Issue delete cleans descendants |
| `@@index([workspaceId, status])` / `[workspaceId, assigneeId]` / `[workspaceId, projectId]` | `issue` | All/Backlog/Blocked/My/Project list hot paths |
| `@@index([issueId, createdAt])` | `issue_history` | Chronological history reads |
| PK `workspaceId` | `workspace_issue_sequence` | One counter per workspace |

---

## 6. Lifecycle semantics at the data layer

All multi-write operations run in a single Prisma `$transaction`. History rows are written in the same transaction as the mutation they describe — a mutation without its history (or vice versa) never commits.

### 6.1 Creation (atomic, spec §3.1)

```text
tx {
  seq = allocate(workspaceId)              // UPDATE sequence RETURNING (D2)
  validate assignee/project/labels same-workspace (D3/D4/D6)
  INSERT issue { workspaceId, seqNumber: seq, title, status: BACKLOG, priority: NO_PRIORITY, blocked: false, ... }
  INSERT issue_label rows for labelIds
  INSERT issue_history { event: CREATED, actor: caller }
  // F6 hook: if assigneeId set → assignment event for notifications (§7)
} → 201 detail (identifier SHIP-{seq})
```

Title is the only required input; `priority`/`status` default when omitted (`status` overridable so boards can create-into-column). Cross-workspace assignee/project/label ids are rejected before insert.

### 6.2 Status change (free, spec §3.2)

`UPDATE issue SET status` any direction + `INSERT history STATUS_CHANGED {old, new}` in the same tx. Kanban cross-column drag is this update with no confirmation; failure rolls back and the UI restores the card; concurrent change is last-write-wins with the client refreshing to the latest row (board contract shared with projects).

### 6.3 Blocked set/clear (orthogonal, spec §3.3)

- **Set:** preconditions `archivedAt IS NULL` + `status ∈ {BACKLOG, TODO, IN_PROGRESS}` + `blocked = false` → `SET blocked = true, blockedReason = <trimmed|null>` + `BLOCKED_SET` history. Rejected on `DONE`/archived.
- **Clear (explicit unblock):** `SET blocked = false, blockedReason = NULL` + `BLOCKED_CLEARED` history. Status untouched.
- **Clear (implicit via Done):** `status → DONE` on a blocked issue atomically clears `blocked/blockedReason` and emits **both** `STATUS_CHANGED` and `BLOCKED_CLEARED` rows. Returning `DONE → active` emits only `STATUS_CHANGED`; blocked stays cleared.

### 6.4 Assignment & planning edits

- **Assign/reassign:** validate target is a current member; `SET assigneeId` + `ASSIGNED`/`UNASSIGNED` history. Same-person set is a no-op (no write, no history, no notification event — spec rule 13).
- **Project attach/detach:** validate same-workspace + `archivedAt IS NULL` on the project; `PROJECT_CHANGED` history. `null` detaches.
- **Priority/due/title:** direct set + corresponding history event (`PRIORITY_CHANGED`, `DUE_DATE_CHANGED`, `TITLE_CHANGED`). Description edits write the column only (no history row — D7).
- **Labels attach/detach:** dedicated mutations (not part of `updateIssue`); each attach/detach validates same-workspace and emits `LABEL_ADDED`/`LABEL_REMOVED`.

### 6.5 Archive / restore / delete (spec §3.6)

- **Archive** (any member, confirmed): `SET archivedAt = now()`. `status`/`blocked` untouched. Emits `ARCHIVED`. Read-only + board-hiding from then on.
- **Restore** (any member, confirmed): `SET archivedAt = NULL`. Returns to stored status + blocked state automatically. Emits `RESTORED`.
- **Delete** (Owner/Admin only, confirmed): one `$transaction`: `DELETE issue` → cascades `issue_label` + `issue_history` (+ F8 comments and F6 notifications per §7 handoff). `workspace_issue_sequence` untouched — the number is never reused. Issues removed from project/cycle counts implicitly (counts are derived queries).

### 6.6 Cross-module unassign on member removal (F3 ↔ F5)

F5 implements the second half of the membership-exit contract (the first half — project transfer — shipped in F4):

```ts
// Owned by issues, called by members inside the caller's transaction.
issuesService.unassignOnMemberExit(
  workspaceId: string,
  userId: string,          // departing member
  tx: Prisma.TransactionClient,
): Promise<number> // count unassigned, for the confirmation dialog
// UPDATE issue SET assigneeId = NULL WHERE workspaceId = ? AND assigneeId = ?
```

Called atomically inside the same `$transaction` that deletes the membership row (remove or leave). Archived issues are included (no `archivedAt` filter) — a departing member's archived assignments are still cleared. Recipient is always "nobody" (unassign, not transfer) — assignment is responsibility, not accountability, unlike project ownership.

### 6.7 Archived-workspace interaction

`workspace.status = ARCHIVED` freezes the container (F2 guard): `GET` allowed, every write rejected with `409 WORKSPACE_ARCHIVED` via `resolveWorkspaceContext({ rejectArchived: true })`. Service reasserts defensively. Archived-issue read-only (`archivedAt` set) is the orthogonal per-row axis, enforced in the issues service.

---

## 7. Forward handoffs — what this model does NOT contain

| Consumer | Contract F5 provides | Landed |
|---|---|---|
| **Projects (F4, already shipped)** | `issue.projectId → project SetNull` + unassign leg in project delete; progress derived from `issue.status` counts | **F5 implements** — closes F4 §6.4/§7 |
| **Notifications (F6)** | Assignment-event hook: on create-with-assignee and on actual assignee change, the issues transaction exposes `{ issueId, workspaceId, newAssigneeId, actorId, identifier }` for `notificationsService.createAssignment(...)` in the same tx. Same-person sets emit nothing. Unassignment emits nothing (spec: no notification for unassignment). | **F5 exposes, F6 implements** — same pattern as F3→F4 Checkpoint B |
| **Cycles (F7)** | `issue.cycleId → cycle.id SetNull` + validation integration + progress derivation. F5 reserves nothing — F7 migrates the column, backfills `null`, and extends `IssueHistoryEvent` with `CYCLE_CHANGED`. | **F7 implements** |
| **Comments (F8)** | `comment.issueId → issue.id Cascade` — issue delete removes its comments (spec rule 9). Archived issues reject new comments (service check on `archivedAt`). | **F8 implements** |
| **Search (F10)** | Generated `tsvector` on `issue(title, description, identifier)` + GIN; grouped search reads issue cards. F5 ships plain `ILIKE` + exact-identifier lookup only. | **F10 implements** |
| **Dashboard (F9)** | Reads issue queries (my/overdue/blocked/recent) + `issue_history` for recent activity; no new issue columns. Recently-viewed recording is an F9 side effect, not an issue-table column. | **F9 implements** |

---

## 8. Migration workflow

Hand-modeled Prisma (like workspace/members/projects):

```bash
# 1 — add IssueStatus, IssuePriority, IssueHistoryEvent enums + Issue, Label,
#     IssueLabel, IssueHistory, WorkspaceIssueSequence models + back-relations
#     on Workspace/User/Project + widen ViewScope with ISSUE
# 2 — add issue.projectId FK (SetNull) — closes the F4 unassign leg
# 3 — run
pnpm --filter @shipyard/api db:migrate -- --name add_issues_labels_history
pnpm --filter @shipyard/api db:generate
```

- The migration produces: 5 tables, 3 new enums, 1 widened enum (`ViewScope`), FKs, and indexes above.
- `label_name_unique` functional index (D6) is appended as raw SQL in the migration folder — same pattern as `project_name_unique`, `workspace_single_owner`, `invitation_single_pending`.
- `ViewScope` widening is additive — zero data rewrite; existing `PROJECT` rows untouched.
- The F1 Testcontainers harness applies migrations automatically each test run.

**Post-migration verification (manual, once):**

```sql
-- no duplicate sequence numbers per workspace
SELECT workspace_id, seq_number, count(*) FROM issue GROUP BY 1,2 HAVING count(*)>1;
-- no duplicate case-insensitive label names per workspace
SELECT workspace_id, lower(name), count(*) FROM label GROUP BY 1,2 HAVING count(*)>1;
-- every issue's sequence has a counter row that is ahead of it
SELECT w.workspace_id, w.next_number, max(i.seq_number) FROM workspace_issue_sequence w
  LEFT JOIN issue i ON i.workspace_id = w.workspace_id GROUP BY 1,2;
```

---

## 9. What we intentionally do NOT model

| Deferred | Why |
|---|---|
| `cycleId` column / relation | Owned by Cycles (F7); F5 omits per Implementation Plan F5 (D5). |
| `notification` table / emission rows | Owned by Notifications (F6); F5 exposes the assignment-event hook only (§7). |
| `comment` table | Owned by Comments (F8); cascade contract fixed here (§7). |
| `tsvector` / full-text indexes | Owned by Search (F10); F5 ships `ILIKE` + exact identifier. |
| Parent-child, dependencies, subtasks, custom types, recurring, templates, watchers, story points, rich text, attachments | Spec §6 out of scope. |
| Manual ordering / rank column | Spec §3.5: same-column drag does not reorder; cards follow active sort. No rank stored. |
| `deletedAt` / trash | Archive covers reversible hiding; delete is confirmed and physical (D1, §6.5). |
| Description history rows | Noise control (D7); description is read from the issue row. |
| Per-workspace identifier prefix column | Constant `SHIP-` for MVP (D2); revisit post-MVP. |

---

## 10. Open product questions — resolved at data layer

| Spec §7 | Decision |
|---|---|
| 1 — priority scale | **Locked:** `NO_PRIORITY / URGENT / HIGH / MEDIUM / LOW`, default `NO_PRIORITY`, rank Urgent→No Priority (D9). |
| 2 — label creation permission | **Any member** may create/rename/recolor/delete labels (D6). Matches issue-write permissions (rule 10). |
| 3 — identifier prefix | **Constant `SHIP-`** + per-workspace sequence (D2). No workspace-derived prefix in MVP. |
| 4 — list pagination | **Cursor pagination** on `(createdAt, id)` / `(seqNumber, id)` depending on sort — specified in `api-design.md`. No offset pagination. |

---

## 11. References

- Shipyard: `features/issues/spec.md`, `features/projects/data-model.md` (archive dimension D1, functional index D3, view_preference D6, F4 §6.4/§7 handoff), `features/members/data-model.md` (partial index pattern, cascade convention, guard chain), `features/workspace/data-model.md` (identity, cascade), `features/auth/data-model.md`, `features/cycles/spec.md` (F7 relation), `features/notifications/spec.md` (F6 hook), `features/comments/spec.md` (F8 cascade), `00-architecture.md` §5/§8/§9, `ADR-001`, `ADR-002`, `Implementation Plan.md` F5
- Prisma indexes & referential actions: `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- PostgreSQL functional indexes: `https://www.postgresql.org/docs/current/indexes-expressional.html`
- PostgreSQL `RETURNING` / atomic counters: `https://www.postgresql.org/docs/current/dml-returning.html`

---

*Next artifact: `api-design.md` — endpoint inventory over `issue` + `label` + `issue_history` (CRUD, lifecycle, labels, history reads, list/board query shapes with cursor pagination), guard chain per route (writes any member, delete Owner/Admin), error codes, and the F6 assignment-event + F3 unassign + F4 project-delete wiring.*
