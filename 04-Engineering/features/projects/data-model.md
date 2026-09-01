# Projects — Data Model

**Status:** Draft for review
**Last updated:** 2026-08-31
**Sources:** `features/projects/spec.md` · `features/workspace/data-model.md` (F2 precedent — cascade convention, identity rules) · `features/members/data-model.md` (F3 precedent — partial indexes, `transferOwnedProjects` contract, guard chain) · `features/auth/data-model.md` (identity — `user` table) · `00-architecture.md` §5, §8, §9 · `ADR-001` (Prisma + Postgres + Better Auth) · `ADR-002` (shared contracts) · `Implementation Plan.md` F4
**Owner:** `apps/api` — Prisma-owned (hand-modeled, like workspace/members).

---

## 1. Overview

Projects owns the **initiative**: a named container that groups related issues toward a shared objective, with an owner, progress, and a lifecycle of its own (spec §1). Projects and Cycles are independent — any project↔cycle connection is derived through issues, never stored (spec rule 10).

One new core table plus one small support table:

| Table | Purpose | Formalized by |
|---|---|---|
| `project` | The initiative: identity, name, operational status, owner, dates, archive lifecycle | **F4 (this milestone)** |
| `view_preference` | Per-user-per-workspace, per-entity List/Kanban view choice (rule 12; shared with Issues F5) | **F4** |

Issue membership and progress are **owned by / derived from Issues (F5)**, not stored here — `project` has no `issue` column and no progress column. The F3 `transferOwnedProjects` contract (already fixed in `members/data-model.md §7`) defines how ownership moves when a member leaves or is removed; F4 implements it on top of this table.

`project.ownerId` references `user.id` (not a membership row id) because the F3 contract operates on user ids — see D2.

---

## 2. Core schema (Prisma-owned)

### 2.1 `project`

One row per initiative. Immutable internal identity via `cuid()`; display name is never an identifier and is not directly unique — uniqueness is enforced by a functional index (D3).

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK `@default(cuid())` | Immutable internal identifier (workspace precedent D3). Used as the URL segment `/w/:slug/projects/:id`. |
| `workspaceId` | `String` | FK → `workspace.id` `onDelete: Cascade` + `@@index([workspaceId])` | Every project belongs to exactly one workspace (spec rule 1). Cascades with the workspace (workspace data-model §7). |
| `name` | `String` | `@db.VarChar(120)` | Display name. Required, trimmed server-side before persist (spec §3.1). Uniqueness per workspace via D3 index. |
| `description` | `String?` | `@db.Text` | Optional free-form blurb (spec §3.1). |
| `status` | `ProjectStatus` | `@default(ACTIVE)` | Operational status `PLANNED \| ACTIVE \| COMPLETED`, free switching in any direction (spec §3.2). Creation lands in `ACTIVE` (spec §3.2, PRD §5.6 "created active"). **No ARCHIVED value** — archive is a separate dimension (D1). |
| `ownerId` | `String` | FK → `user.id` `onDelete: Restrict` + `@@index([ownerId])` | Project Owner = accountability only, grants no permissions (spec rule 3). Always a current workspace member (service invariant, D2). |
| `archivedAt` | `DateTime?` | `@@index([archivedAt])` | Presence ⇒ archived (read-only, hidden from boards/lists). **Kept** on restore to preserve history; the operational `status` is untouched, so restore returns to the prior status for free (D1). |
| `startDate` | `DateTime?` | `@db.Date` | Optional, day-precision (D4). |
| `targetDate` | `DateTime?` | `@db.Date` | Optional, day-precision (D4). |
| `createdAt` | `DateTime` | `@default(now())` | |
| `updatedAt` | `DateTime` | `@updatedAt` | Renaming never breaks references — references use `id`, not `name` (workspace rule 10 precedent). |

> No soft-delete flag, no `deletedAt`: permanent delete physically removes the row in one transaction (§6.3). Archive ≠ delete — archive is `archivedAt` set; delete removes the row and clears every issue's project reference in the same transaction (see §7 for the F5 handoff).

> No `progress` column — progress is derived from issues (spec §3.4, rule 11) and is never manually set. Landed when Issues ships (F5); see §7.

```prisma
enum ProjectStatus {
  PLANNED
  ACTIVE
  COMPLETED
}

model Project {
  id          String        @id @default(cuid())
  workspaceId String
  name        String        @db.VarChar(120)
  description String?       @db.Text
  status      ProjectStatus @default(ACTIVE)
  ownerId     String
  archivedAt  DateTime?
  startDate   DateTime?     @db.Date
  targetDate  DateTime?     @db.Date
  createdAt   DateTime      @default(now())
  updatedAt   DateTime      @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  owner     User      @relation(fields: [ownerId], references: [id], onDelete: Restrict)

  @@index([workspaceId])
  @@index([status])
  @@index([ownerId])
  @@index([archivedAt])
  @@map("project")
}
```

Uniqueness constraint (applied as raw SQL in the migration, D3):

```sql
-- workspace-scoped, case-insensitive, trimmed-at-write; deleted rows are physically gone
CREATE UNIQUE INDEX project_name_unique
  ON project (workspace_id, lower(name));
```

### 2.2 `view_preference` — per-user per-workspace, per-entity view choice (rule 12, shared with Issues)

Spec rule 12: the list/Kanban view is stored per user per workspace, independent of the issues preference; list-only subviews never overwrite it; List is the default on first visit.

This is a **generic** per-user-per-workspace-entity preference: one `scope` row per user per workspace per entity. It ships here for Projects (`PROJECT`) and is deliberately shaped so Issues (`ISSUE`, F5) and any future board-bearing entity can reuse it — a single table, additive `ViewScope` widening, no second table for Issues. A row exists **only** once the user makes an explicit selection for a given `(workspace, user, scope)` — absence of a row equals that scope's List default.

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK `@default(cuid())` | |
| `workspaceId` | `String` | FK → `workspace.id` `onDelete: Cascade` + `@@index([workspaceId])` | Preference scoped to a workspace (rule 12: per per workspace, never global). |
| `userId` | `String` | FK → `user.id` `onDelete: Cascade` | The user the preference belongs to (rule 12: per user). |
| `scope` | `ViewScope` | — | Which entity the view applies to: `PROJECT` this milestone; `ISSUE` added by F5 (additive enum widening, D6). |
| `view` | `ViewType` | `@default(LIST)` | `LIST \| KANBAN`. |
| `updatedAt` | `DateTime` | `@updatedAt` | Selecting a view bumps this; toggling issues never touches a `PROJECT` row (independence, rule 12). |

```prisma
enum ViewScope {
  PROJECT
  // F5 adds: ISSUE
}

enum ViewType {
  LIST
  KANBAN
}

model ViewPreference {
  id          String    @id @default(cuid())
  workspaceId String
  userId      String
  scope       ViewScope
  view        ViewType  @default(LIST)
  updatedAt   DateTime  @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([workspaceId, userId, scope]) // one preference per user per workspace per entity
  @@index([workspaceId])
  @@map("view_preference")
}
```

> F5 does **not** add a second preference table. It only widens `ViewScope` with `ISSUE` (additive, zero data rewrite) and starts reading/writing the same rows with `scope = ISSUE`. The `PROJECT` rows are untouched by issue view toggles, preserving rule 12's independence.

`Workspace` and `User` gain the back-relation fields (schema edits required on both existing models):

```prisma
model Workspace {
  // ... existing F2/F3 fields
  projects        Project[]
  viewPreferences ViewPreference[]
}

model User {
  // ... existing F1/F3 fields
  ownedProjects   Project[]
  viewPreferences ViewPreference[]
}
```

---

## 3. Key decisions & alternatives

### D1 — Archive is a separate dimension (`archivedAt`), not an enum value

**Decision:** `status` holds only the three operational values (`PLANNED | ACTIVE | COMPLETED`); archiving is expressed purely by `archivedAt` being set (plus a read-only guard at the service layer). This differs deliberately from Workspace, where `status` carries `ARCHIVED`.

*Reason:* the Projects spec requires archive to be **reversible with full preservation of the operational status** — "Restore returns it to the status it held before archiving" (spec §3.2, rule 8). A single enum with `ARCHIVED` would force us to remember the prior operational value in a second column or re-derive it. With a separate dimension, archive = set `archivedAt` (operational status untouched) and restore = clear `archivedAt`. The status control never has to exclude `ARCHIVED` (§3.2, PRD §5.6) because `ARCHIVED` simply isn't in the enum. Board/board-hiding is `WHERE archivedAt IS NULL`.

*Considered and rejected:* a `Boolean archived` + `archivedAt` — the boolean is redundant; `archivedAt` presence already signals archived, and filtering/indexing on a nullable timestamp is straightforward.

### D2 — `ownerId` references `user.id`, not a membership row; "current member" is a service invariant

**Decision:** `project.ownerId` is an FK to `user.id` (`onDelete: Restrict`) + `@@index([ownerId])`. The invariant "the Owner is always a current member of the project's workspace" (spec rule 2) is enforced at the **service layer** (via the members guard chain + the F3 `transferOwnedProjects` contract), not by a DB FK.

*Reason:* the already-fixed F3 contract operates on **user ids** — `transferOwnedProjects(workspaceId, fromUserId, toOwnerUserId, tx)` (members `data-model.md §7`). Referencing `user.id` lets F4 implement that contract verbatim without a join-to-membership indirection. A composite "user is member of same workspace" constraint cannot be a simple FK anyway (a user belongs to many workspaces), so the FK to `user` is the strongest atomic guarantee available and the membership invariant is a documented service contract.

`onDelete: Restrict` (rather than `Cascade`/`SetNull`): deleting a user who owns projects must fail at the DB until ownership is transferred — orphaned projects (no owner) are never allowed (spec rule 2: exactly one owner). Membership rows cascade on user delete (F3), but `project.ownerId → user` is `Restrict`, so the ordering is: transfer ownership first, then delete. No user-deletion path exists in MVP; `Restrict` is the safe default that forces correct sequencing.

### D3 — Case-insensitive workspace-scoped name uniqueness via raw functional index

**Decision:** Uniqueness is a functional index the Prisma schema language cannot express:

```sql
CREATE UNIQUE INDEX project_name_unique ON project (workspace_id, lower(name));
```

Names are trimmed server-side before persist (spec §3.1). Because permanent delete physically removes the row (no soft delete), every non-deleted row participates — archived rows correctly continue to **reserve** their name (rule 4), and deleting frees it for reuse automatically.

*Considered and rejected:* (a) `@@unique([workspaceId, name])` — case-sensitive, would allow `"API"` and `"api"` to coexist, violating spec rule 4; (b) a denormalized `nameKey` column with `@@unique([workspaceId, nameKey])` for native Prisma expression — works, but adds a column that must be kept in sync on every rename; the functional index is the same pattern the workspace/members milestones already use for partial indexes (raw SQL in the migration folder) and avoids the sync risk.

### D4 — Date-only fields use `@db.Date`

**Decision:** `startDate` and `targetDate` are day-precision (`@db.Date` maps to Postgres `DATE`). Projects are planned at day granularity (target date = a date, not a timestamp); using `DATE` avoids timezone-shift bugs when the web serializes/deserializes day strings and makes "filter by start/target date" (PRD §5.6) a plain comparison. The web contract exposes these as `YYYY-MM-DD` strings.

### D6 — Generic `view_preference` table shared with Issues, keyed by `scope`

**Decision:** One `view_preference` table keyed by `(workspaceId, userId, scope)` where `scope` is an enum (`PROJECT` now, `ISSUE` added by F5). Projects and Issues both need an identical per-user-per-workspace List/Kanban choice, stored independently (rule 12), with List as the absent-row default. Rather than a `project_view_preference` table now and a parallel `issue_view_preference` later, a single scoped table grows additively — F5 widens `ViewScope`, no new table, no migration of existing rows.

*Considered and rejected:* (a) separate per-feature tables — duplicates the exact same shape and the independence/upsert logic for no benefit; (b) a generic JSON `preferences` column on a membership/user row — hides the shape from the schema and loses the DB-level `@@unique([workspaceId, userId, scope])` guarantee; (c) client-side localStorage — the precedence in the spec ("stored", "per workspace") and cross-device expectations favor server persistence.

### D5 — No project `slug`; route by `cuid()` id

**Decision:** Project URLs are `/w/:workspaceSlug/projects/:projectId` where `:projectId` is the `cuid()`. Unlike the workspace switcher (F2 D3), there is no short stable token need — project links are deep links reached from a board/list, not typed or switched frequently.

*Considered and rejected:* a generated `slug` like Workspace — adds a generated column, uniqueness, and collision-retry complexity for no MVP need; names must not be identifiers (rule 4-style reasoning), so a slug would have to be separately generated anyway. Revisit post-MVP if project URLs need to be human-readable/shareable.

---

## 4. Shared contracts (`packages/shared`)

Added in F4, consumed by `api` and `web` (ADR-002). All schemas mirror the Prisma enums above.

```ts
// zod enums — mirror Prisma enums §2
// projectStatusSchema: PLANNED | ACTIVE | COMPLETED
// viewScopeSchema / viewTypeSchema are generic (shared with Issues F5):
export const viewScopeSchema = z.enum(["PROJECT"]); // F5 adds "ISSUE"
export const viewTypeSchema = z.enum(["LIST", "KANBAN"]);

// canonical name bound — matches VarChar(120); trimmed
export const projectNameSchema = z.string().trim().min(1).max(120);

// request contracts owned by the projects module
export const createProjectSchema = z.object({
  name: projectNameSchema,
  description: z.string().max(10000).optional(),
  startDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(), // YYYY-MM-DD
  targetDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(),
});

export const updateProjectSchema = z.object({
  name: projectNameSchema.optional(),
  description: z.string().max(10000).nullable().optional(),
  status: projectStatusSchema.optional(),        // operational fields
  startDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).nullable().optional(),
  targetDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).nullable().optional(),
});

export const transferProjectOwnerSchema = z.object({
  targetMemberId: z.string().cuid(), // membership row id in the same workspace, per F3 members convention
});

export const setViewPreferenceSchema = z.object({
  scope: viewScopeSchema, // PROJECT now; ISSUE in F5
  view: viewTypeSchema,
});

// response contracts
export const projectOwnerCardSchema = z.object({
  memberId: z.string(), // membership row id (from the join)
  userId: z.string(),
  name: z.string(),
  email: z.string().email(),
  image: z.string().nullable(),
});

export const projectCardSchema = z.object({
  id: z.string(),
  workspaceId: z.string(),
  name: z.string(),
  status: projectStatusSchema,
  owner: projectOwnerCardSchema,
  startDate: z.string().nullable(), // YYYY-MM-DD
  targetDate: z.string().nullable(), // YYYY-MM-DD
  archivedAt: z.string().datetime().nullable(),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});

export const projectDetailSchema = projectCardSchema.extend({
  description: z.string().nullable(),
});
```

`view_preference` helpers round out the module (`getViewPreference(workspaceId, userId, scope)` → `ViewType`, `setViewPreference(workspaceId, userId, scope, view)`); absence of a row resolves to `LIST` (rule 12 default). Because the table is generic, these helpers take `scope` and are reused verbatim by Issues in F5.

---

## 5. Integrity invariants → spec rule mapping

| Spec rule | Enforcement point |
|---|---|
| 1 — every project in exactly one workspace | `workspaceId` FK `Cascade` + index |
| 2 — exactly one Owner, always a current member | `ownerId` FK `Restrict` (atomic non-null) + service membership invariant via F3 `transferOwnedProjects` contract (D2) |
| 3 — ownership grants no permissions | no role/permission derived from `ownerId`; all write gates read `WorkspaceRole` from membership (F3 matrix) |
| 4 — name unique per workspace, archived reserve, delete releases | D3 functional index `(workspace_id, lower(name))` over all non-deleted rows |
| 5 — Owner/Admin transfer non-archived project | `transferProjectOwnerSchema` target validated as same-workspace member; `archivedAt IS NULL` precondition at service |
| 6 — on member remove/leave, owned projects → Workspace Owner (archived included) | F3 Checkpoint B `transferOwnedProjects(workspaceId, fromUserId, toOwnerUserId, tx)` in the same `$transaction`; recipient resolved as current `OWNER`, never caller-supplied |
| 7 — operational statuses switch freely; Archived separate | `status` enum has only PLANNED/ACTIVE/COMPLETED; archive = `archivedAt` dimension (D1) |
| 8 — restore returns to prior operational status | `archivedAt` cleared; `status` untouched by archive/restore |
| 9 — delete permanent, never deletes issues, clears assignment atomically | row physically deleted + issue `projectId` cleared in one `$transaction` (F5 handoff, §7) |
| 10 — project↔cycle derived through issues, never stored | no `cycleId` on `project`; no `projectId` on `cycle` |
| 11 — progress derived, blocked not counted, never manual | no progress column; computed from issue statuses in F5 |
| 12 — view pref per user per workspace, independent, List default | `view_preference` table, `@@unique([workspaceId, userId, scope])`, row-absent ⇒ LIST; `PROJECT` rows independent of `ISSUE` rows (F5) |

Integrity summary — constraints added or relied upon in F4:

| Constraint | Where | Purpose |
|---|---|---|
| `project_name_unique` functional unique | `project (workspace_id, lower(name))` | Case-insensitive name uniqueness per workspace |
| FK `project.ownerId → user` `Restrict` | `project` | Exactly one non-null owner; block user delete until ownership transferred |
| FK `project.workspaceId → workspace` `Cascade` | `project` | Project dies with its workspace |
| `@@unique([workspaceId, userId, scope])` | `view_preference` | One view preference per user per workspace per entity |
| `@@index([workspaceId])` / `[status]` / `[ownerId]` / `[archivedAt]` | `project` | List/board/filter hot paths |
| FK cascades (workspace → children, user → preference) | both tables | Workspace delete + user delete cleanly remove owned scoped rows |

---

## 6. Lifecycle semantics at the data layer

### 6.1 Creation (atomic, spec §3.1)

Single transaction inserts `project` with `status = ACTIVE` and `ownerId = creatorUserId` (`creator becomes the Owner`, spec rule). The creator is guaranteed a current member (route is member-gated), so the service invariant holds at creation. Name uniqueness is enforced at commit by D3.

### 6.2 Status change (free, spec §3.2)

`UPDATE project SET status = <PLANNED|ACTIVE|COMPLETED>` — any direction, no confirmation. Never includes archived (not an enum value). Cross-column on the Kanban board is just this status update plus membership write-permission at the service layer; a failed drag rolls the UI back, a concurrent change re-reads the latest row (board contract shared with issues).

### 6.3 Archive / restore (reversible, spec §3.2)

- **Archive** (Owner/Admin, confirmed): `SET archivedAt = now()`. Operational `status` **unchanged**. Read-only + board-hiding (`WHERE archivedAt IS NULL`) enforced at service layer.
- **Restore** (confirmed): `SET archivedAt = NULL`. Returns to the stored operational `status` automatically. History (the archived period) is preserved by the fact that we don't delete anything; `updatedAt` bumps.

### 6.4 Permanent delete (irreversible, spec §3.2/rule 9)

Preconditions (service): caller is Owner/Admin, and the verbose `confirmName` body echoes the exact current name.

Runs in **one `$transaction`**: delete the `project` row **and** clear `projectId` on every issue currently assigned to it (`UPDATE issue SET projectId = NULL WHERE projectId = ?`). If either fails, neither persists — issues are never orphaned-to-deleted and never deleted themselves. Until Issues (F5) exists, this is simply the row delete; the unassign leg is wired in when `issue.projectId` lands (§7).

### 6.5 Ownership transfer (spec §3.3)

`transferProjectOwnerSchema { targetMemberId }` → service resolves the target membership row to its `userId`, verifies same workspace, still present, `archivedAt IS NULL`, and `target != owner`, then `UPDATE project SET ownerId = <targetUserId>`. Runs inside a transaction. Recipient's workspace role is never touched (rule 3).

### 6.6 Cross-module transfer on member removal (F3 Checkpoint B ↔ F4)

F4 implements `transferOwnedProjects(workspaceId, fromUserId, toOwnerUserId, tx)` per the contract fixed in `members/data-model.md §7`:

- `UPDATE project SET ownerId = toOwnerUserId WHERE workspaceId = ? AND ownerId = fromUserId` — this naturally covers **archived** projects too (no `archivedAt` filter), satisfying spec rule 6.
- Called inside the same `$transaction` that deletes the membership row; all-or-nothing.

---

## 7. Issues (F5) handoff — what this model does NOT contain

- **No `issue` relation / `projectId` on issues yet.** Issues are owned by F5. F5 adds `Issue.projectId? → Project` (FK, `onDelete: SetNull`) plus the unassign leg in `project` delete (§6.4) and the progress derivation (spec §3.4).
- **No `cycle` relation.** project↔cycle is derived through shared issues (rule 10) — nothing stored in F4.
- `project` ships in F4 with only the fields above; the workspace cascade container already reserves `project` in `workspace/data-model.md §7` (`onDelete: Cascade`).

---

## 8. Migration workflow

Hand-modeled Prisma (like workspace/members):

```bash
# 1 — add ProjectStatus, ViewScope, ViewType enums + Project, ViewPreference models + back-relations
# 2 — run
pnpm --filter @shipyard/api db:migrate -- --name add_projects_and_view_preference
pnpm --filter @shipyard/api db:generate
```

- The migration produces: 2 tables (`project`, `view_preference`), 3 enums (`ProjectStatus`, `ViewScope`, `ViewType`), the FKs, and indexes above.
- The functional unique name index (D3) cannot be expressed in Prisma schema, so it's appended as raw SQL in the migration folder — same pattern as `workspace_single_owner` and `invitation_single_pending` in prior milestones.
- `ViewScope` is widened with `ISSUE` in F5 — additive, zero data rewrite; the same `view_preference` table is reused (D6).
- The F1 Testcontainers harness applies migrations automatically each test run.

**Post-migration verification (manual, once):**

```sql
-- no duplicate case-insensitive names per workspace
SELECT workspace_id, lower(name), count(*) FROM project GROUP BY 1,2 HAVING count(*)>1;
```

---

## 9. What we intentionally do NOT model

| Deferred | Why |
|---|---|
| `issue` relation / progress column | Owned by Issues (F5); progress is derived, never stored (rule 11). |
| `cycle` relation | Derived through issues (rule 10). |
| Project `slug` | D5 — route by `cuid()` id in MVP. |
| Templates, milestones, dependencies, roadmaps, health indicators, custom fields, cross-project reporting | Spec §6 out of scope. |
| Budget/resource fields | Spec §6 out of scope. |
| `createdById` | Creator == first Owner and ownership is reassignable; no product need to persist original creator in MVP. Activity history (spec open Q1) can add it if required. |
| Soft delete / trash | Archive covers reversible hiding; delete is double-confirmed and physical (D1, §6.4). |

---

## 10. Open product questions — resolved at data layer

| Spec §7 | Decision |
|---|---|
| 1 — activity granularity | Not a data-model concern for the core table; if action-level activity ships, it will be an Issue-style activity table, decided at implementation. `createdAt`/`updatedAt` already cover record-level display. |
| 2 — list pagination | **No pagination.** Projects are few per workspace; the list/board reads all non-archived rows. A `LIMIT` guard may be added at the API layer for safety, not as model structure. |
| 3 — name normalization | **Functional lower() index (D3)** + trim-at-write. Case-insensitive and workspace-scoped, satisfied at the DB layer. |

---

## 11. References

- Shipyard: `features/projects/spec.md`, `features/workspace/data-model.md` (F2 precedent), `features/members/data-model.md` (F3 precedent — `transferOwnedProjects` contract §7, partial index pattern, guard chain), `features/auth/data-model.md`, `00-architecture.md`, `ADR-001`, `ADR-002`, `Implementation Plan.md` F4
- Prisma indexes & referential actions: `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- PostgreSQL functional indexes: `https://www.postgresql.org/docs/current/indexes-expressional.html`

---

*Next artifact: `api-design.md` — endpoint inventory over `project` + `view_preference`, guard chain per route (create/edit Owner/Admin vs view all members), error codes, and the F3 Checkpoint B transfer wiring.*
