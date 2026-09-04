# Activity Log — Data Model

**Status:** Draft for review
**Last updated:** 2026-09-04
**Sources:** `features/activity/spec.md` · `features/issues/data-model.md` (F5 precedent — typed events, `SetNull` actor, cascade convention) · `features/comments/data-model.md` (F8 precedent — snapshot-vs-join reasoning, same-tx side effects) · `features/notifications/data-model.md` (F6 precedent — internal-only emission, inverted cascade reasoning) · `features/members/data-model.md` (F3 precedent — membership vs user rows) · `features/workspace/data-model.md` (F2 precedent — identity, cascade) · `features/auth/data-model.md` (identity — `user.name`) · `00-architecture.md` §5, §8, §9 · `ADR-001` (Prisma + Postgres) · `ADR-002` (shared contracts) · `Implementation Plan.md` F9 §7 (dashboard migrates its panel onto this feed)
**Owner:** `apps/api` — Prisma-owned (hand-modeled, like all prior features).

---

## 1. Overview

Activity Log owns the **workspace narrative** as stored rows: one immutable, snapshot-based event per main action across workspace/members/projects/issues/comments/cycles. It is deliberately the inverse of notifications: where notification rows cascade with their source (dead refs are bad there), activity rows outlive their source (a deleted entity's story is the point).

One new table, two new enums:

| Table / Change | Purpose | Formalized by |
|---|---|---|
| `activity_event` | The frozen event: scope, actor (+frozen name), kind, polymorphic target (+frozen title), rendered summary | **This milestone** |
| `ActivityKind` | 30 typed kinds across six areas (additive for future areas) | **This milestone** |
| `ActivityEntityType` | `WORKSPACE \| MEMBER \| INVITATION \| PROJECT \| ISSUE \| COMMENT \| CYCLE` | **This milestone** |

Per-entity histories stay authoritative for detail views (`issue_history`, invitation `status`, comment `editedAt`). Issue/comment actions dual-write both rows in one transaction — no consolidation, no replacement.

---

## 2. Core schema (Prisma-owned)

### 2.1 `activity_event`

One row per main event. Immutable — no update path, no delete path except source cascades below. Target references are **plain strings, never foreign keys** (D3): deleting an issue/project/cycle/comment/invitation touches zero log rows.

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK `@default(cuid())` | Immutable internal identifier. Cursor tiebreak with `createdAt`. |
| `workspaceId` | `String` | FK → `workspace.id` `onDelete: Cascade` + `@@index([workspaceId])` | Exactly one workspace (spec rule 1). Workspace delete removes its log (rule 6). |
| `actorId` | `String?` | FK → `user.id` `onDelete: SetNull` + `@@index([actorId])` | Who acted. `SetNull` on user delete — row survives on the frozen `actorName` (rule 6, D4). |
| `actorName` | `String` | `@db.VarChar(255)` | Frozen display name at emit (spec Q3). Rendered verbatim forever. Member display name normally; **invitee email** for invitation-lifecycle rows (`MEMBER_INVITED/JOINED/DECLINED/REVOKED`) — the invitee may never become a member, and email matches the invitation row (D4). |
| `kind` | `ActivityKind` | `@@index([kind])` | Typed event (§2.2). Area filters map to kind sets server-side. |
| `entityType` | `ActivityEntityType` | — | What the event is about (navigation mapping). |
| `entityId` | `String?` | — | Target row id at emit (no FK — survives deletion, D3). `null` for workspace-level events addressing the workspace itself. |
| `entityTitle` | `String?` | `@db.Text` | Frozen title at emit: issue `"SHIP-24 · Fix login"`, project/cycle name, invitee email, member name. `null` only where the summary alone suffices. |
| `summary` | `String` | `@db.Text` | Frozen rendered sentence, e.g. `"Maya moved SHIP-24 from Todo to In Progress"`. Never rewritten (rule 3). |
| `createdAt` | `DateTime` | `@default(now())` | Event order — page is newest-first. No `updatedAt` (immutable). |

```prisma
enum ActivityKind {
  WORKSPACE_CREATED
  WORKSPACE_UPDATED
  WORKSPACE_ARCHIVED
  WORKSPACE_RESTORED
  MEMBER_INVITED
  MEMBER_JOINED
  MEMBER_DECLINED
  MEMBER_INVITE_REVOKED
  MEMBER_REMOVED
  MEMBER_LEFT
  MEMBER_ROLE_CHANGED
  OWNERSHIP_TRANSFERRED
  PROJECT_CREATED
  PROJECT_RENAMED
  PROJECT_STATUS_CHANGED
  PROJECT_OWNER_TRANSFERRED
  PROJECT_ARCHIVED
  PROJECT_RESTORED
  PROJECT_DELETED
  ISSUE_CREATED
  ISSUE_STATUS_CHANGED
  ISSUE_ASSIGNED
  ISSUE_BLOCKED_SET
  ISSUE_BLOCKED_CLEARED
  ISSUE_ARCHIVED
  ISSUE_RESTORED
  ISSUE_DELETED
  COMMENT_CREATED
  COMMENT_DELETED
  CYCLE_CREATED
  CYCLE_STARTED
  CYCLE_COMPLETED
  CYCLE_REOPENED
  CYCLE_ARCHIVED
  CYCLE_RESTORED
  CYCLE_DELETED
}

enum ActivityEntityType {
  WORKSPACE
  MEMBER
  INVITATION
  PROJECT
  ISSUE
  COMMENT
  CYCLE
}

model ActivityEvent {
  id          String              @id @default(cuid())
  workspaceId String
  actorId     String?
  actorName   String              @db.VarChar(255)
  kind        ActivityKind
  entityType  ActivityEntityType
  entityId    String?
  entityTitle String?             @db.Text
  summary     String              @db.Text
  createdAt   DateTime            @default(now())

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  actor     User?     @relation(fields: [actorId], references: [id], onDelete: SetNull)

  @@index([workspaceId])
  @@index([workspaceId, createdAt])
  @@index([kind])
  @@index([actorId])
  @@map("activity_event")
}
```

`Workspace` gains `activityEvents ActivityEvent[]`. `User` gains `activityActions ActivityEvent[]`. No back-relations from issue/project/cycle/comment/invitation — there are no FKs to relate (D3).

No raw-SQL indexes needed: no partial or functional indexes (unread-style subsets don't exist — no read state on immutable rows).

---

## 3. Key decisions & alternatives

### D1 — One table + kind enum (not per-area tables)

**Decision:** all six areas share one table branched by `kind`, mirroring notifications D1. Every area's lifecycle is identical (emit in-tx → newest-first page → survive source → die with workspace). Per-area tables would sextuple the page query and the prune/retention story for zero MVP divergence. New areas widen the enum additively.

### D2 — Emission is strict, synchronous, in-source-tx (locked)

**Decision:** each source service calls `activityService.record(event, tx)` inside its own `$transaction`; a failed insert rolls back the source action (spec Q7). Rationale: a narrative log with silent holes lies quietly — worse than a failed write with an honest error. Same atomicity as notifications rule 7, opposite of the dashboard trail's best-effort recording (a nicety vs a record).

### D3 — Targets are plain strings + frozen snapshots, never FKs (locked)

**Decision:** `entityId`/`entityTitle` carry no referential constraint, so `ISSUE_DELETED` / `MEMBER_DECLINED` / `PROJECT_DELETED` rows survive the very deletion they describe (spec rule 5). Navigation resolves `entityType + entityId` against live rows at read time; misses render `entityTitle` as frozen text with no link (never 404 the page).

*Considered and rejected:* FK-per-target columns (nullable `issueId`, `projectId`, … with `Cascade`) — the delete-event row would cascade-delete itself; `SetNull` variants would keep a hollow row while still requiring six nullable FKs plus fallback rendering anyway. Plain strings with snapshots dominate on both simplicity and correctness.

### D4 — Actor frozen name + `SetNull` link (locked)

**Decision:** `actorName` frozen at emit; `actorId` nulled on user delete (spec Q3/Q6). Reads render `actorName` verbatim; a non-null `actorId` additionally links to the member context where the membership still exists. Leaving a workspace changes nothing (membership ≠ user row).

**Non-member actor convention (locked):** invitation-lifecycle rows (`MEMBER_INVITED/JOINED/DECLINED/REVOKED`) identify the invitee by **email** in `actorName` — the invitee acts (decline/join) without holding membership, and may never gain it. `actorId` is set whenever the acting side has a user row (inviter on invite/revoke; authenticated invitee on decline/join — decline requires a verified session) and stays `null` only where no account acted. Email is stable across the invite→decline→join arc: it matches the invitation row and survives account renames.

### D5 — Snapshots frozen at emit, summary pre-rendered (locked)

**Decision:** `summary` is composed by the emitting service from frozen values (`"Owner invited bob@example.com as Member"`, `"Maya moved SHIP-24 from Todo to In Progress"`). Later renames, reassignments, or deletions never touch the row (rule 3). The page renders `summary` verbatim plus structured chips (kind area, time) — no client copy presenter needed, unlike notifications.

### D6 — No backfill, no retention job (locked)

**Decision:** the table starts empty at deploy (spec Q6); pre-existing issues/comments/projects have no log rows and the page makes no claim otherwise. No TTL, no prune cap (spec Q5) — rows are two short texts plus ids; growth is linear in main events, revisited with evidence.

### D7 — Dual-write with per-entity histories, no consolidation

**Decision:** issue/comment actions write BOTH the entity history (`issue_history` row, comment row) AND the activity row in the same tx. The entity tables stay authoritative for detail timelines (per-issue view keeps its exact F5 shape); the log is the cross-entity narrative (spec rule 7). Deleting the entity removes its history rows per existing cascades but keeps its log rows per D3 — each serves its reader.

---

## 4. Shared contracts (`packages/shared`)

Added here, consumed by `api` and `web` (ADR-002). Event (internal) and card (HTTP) shapes are distinct — events are service-call arguments, cards are page responses.

```ts
// zod enums — mirror Prisma enums §2
export const activityKindSchema = z.enum([ /* 36 kinds, §2.1 */ ]);
export const activityEntityTypeSchema = z.enum([
  "WORKSPACE", "MEMBER", "INVITATION", "PROJECT", "ISSUE", "COMMENT", "CYCLE",
]);
export const activityAreaSchema = z.enum([
  "workspace", "members", "projects", "issues", "comments", "cycles",
]); // page filter → kind sets, server-side mapping

// internal event contract (service-to-service, never HTTP bodies — D2)
export const recordActivityEventSchema = z.object({
  workspaceId: z.string(),
  actorId: z.string(),
  // actorName: member display name — or invitee EMAIL for invitation-lifecycle
  // kinds (MEMBER_INVITED/JOINED/DECLINED/REVOKED); the invitee may never be a
  // member, and email matches the invitation row (D4).
  actorName: z.string().max(255),
  kind: activityKindSchema,
  entityType: activityEntityTypeSchema,
  entityId: z.string().nullable().optional(),
  entityTitle: z.string().nullable().optional(),
  summary: z.string().min(1).max(2000),
});

// response contracts (HTTP reads — frozen rows rendered verbatim, D5)
export const activityEventCardSchema = z.object({
  id: z.string(),
  workspaceId: z.string(),
  actorId: z.string().nullable(),
  actorName: z.string(),
  kind: activityKindSchema,
  area: activityAreaSchema, // derived from kind (filter chips, icons)
  entityType: activityEntityTypeSchema,
  entityId: z.string().nullable(),
  entityTitle: z.string().nullable(),
  summary: z.string(),
  createdAt: z.string().datetime(),
});

// page (newest-first cursor over (createdAt, id) DESC)
export const activityListQuerySchema = z.object({
  area: activityAreaSchema.optional(),                 // omitted ⇒ all areas
  actorId: z.string().cuid().optional(),
  entityType: activityEntityTypeSchema.optional(),
  limit: z.coerce.number().int().min(1).max(100).optional(), // default 25
  cursor: z.string().optional(),                        // opaque base64url of (createdAt, id)
});

export const activityListPageSchema = z.object({
  events: z.array(activityEventCardSchema),
  nextCursor: z.string().nullable(), // null ⇒ end
});
```

---

## 5. Integrity invariants → spec rule mapping

| Spec rule | Enforcement point |
|---|---|
| 1 — one workspace, one named actor per event | `workspaceId` non-null FK; `actorId` + frozen `actorName` (member name, or invitee email for invitation rows) (D4) |
| 2 — strict synchronous recording | `record(event, tx)` inside source tx only; no standalone/best-effort path (D2) |
| 3 — immutable snapshots | No update route; no service mutator; renames/deletes touch zero rows (D3/D5) |
| 4 — any member reads all | member-only guard, no authorship/role layer (api-design) |
| 5 — source deletion keeps rows | No target FKs exist to cascade (D3) |
| 6 — workspace delete clears; user delete nulls actor | `workspaceId Cascade` / `actorId SetNull` (§2) |
| 7 — entity histories stay authoritative | Dual-write, no migration of `issue_history`/invitation readers (§6.x, D7) |

Integrity summary — constraints added here:

| Constraint | Where | Purpose |
|---|---|---|
| FK `activity_event.workspaceId → workspace` `Cascade` | `activity_event` | Log dies with its workspace |
| FK `activity_event.actorId → user` `SetNull` | `activity_event` | Survives actor deletion on frozen name |
| `@@index([workspaceId])` / `[workspaceId, createdAt]` | `activity_event` | Page walk hot path |
| `@@index([kind])` / `[actorId]` | `activity_event` | Area + actor filter hot paths |

---

## 6. Lifecycle semantics at the data layer

### 6.1 Emission sites (one row per main action, same-tx)

Each source service composes `summary` from frozen values and calls `record()` before commit. Representative sites (full per-route wiring in `api-design.md`):

```text
workspace rename:  record(WORKSPACE_UPDATED, entity: WORKSPACE/slug, "Maya renamed the workspace to Harbor")
invite:            record(MEMBER_INVITED, entity: INVITATION/id, "Owner invited bob@example.com as Member")
decline:           record(MEMBER_DECLINED, entity: INVITATION/id, actor: invitee userId/email, "bob@example.com declined the invite")
remove/leave:      record(MEMBER_REMOVED / MEMBER_LEFT, entity: MEMBER/userId, "...")
project create:    record(PROJECT_CREATED, entity: PROJECT/id, "Maya created project Ship Payroll")
issue move:        record(ISSUE_STATUS_CHANGED, entity: ISSUE/id, "Maya moved SHIP-24 from Todo to In Progress")
comment post:      record(COMMENT_CREATED, entity: COMMENT/id, "Maya commented on SHIP-24")
cycle start:       record(CYCLE_STARTED, entity: CYCLE/id, "Owner started Sprint 13")
issue delete:      record(ISSUE_DELETED, entity: ISSUE/id, "Owner deleted SHIP-24 · Fix login") ← survives via D3
```

Edits to comments, label ops, priority-only changes, resends, view-preference, and notification interactions emit nothing (spec §3.1 exclusion list).

### 6.2 Reads (page walk)

`WHERE workspaceId [AND kind IN area-set] [AND actorId = ?] [AND entityType = ?]`, order `(createdAt DESC, id DESC)`, cursor over last row, default 25 / max 100. Navigation targets resolve live per row (`entityType + entityId`); misses render frozen `entityTitle` without links. Archived workspaces read identically (log frozen by upstream silence, not by guard).

### 6.3 Cascades (what deletes what)

| Source delete | Effect |
|---|---|
| Workspace delete | all its rows gone (`workspaceId Cascade`) |
| Issue / project / cycle / comment / invitation delete | **nothing** — rows survive with snapshots (D3, rule 5) |
| Actor user delete | `actorId` nulled, `actorName` + rows intact (D4) |
| Member leave/remove | nothing (membership ≠ user row; actor link + name intact) |

### 6.4 Dashboard migration (F9 §7 consumer)

On landing, dashboard Recent Activity swaps its derivation (§6.4 of dashboard data-model) for `activity_event WHERE workspaceId AND kind IN (issue/comment kinds) ORDER DESC LIMIT 20`. Pre-log history stays visible only via entity detail timelines (accepted gap, spec Q6). Panel bound and card mapping stay unchanged; `dashboardActivityKindSchema` extends with the new kinds only if the panel ever shows them (it doesn't in MVP — panel keeps issue/comment kinds).

---

## 7. Forward handoffs

| Consumer | Contract provided | Landed |
|---|---|---|
| **Dashboard (F9)** | `listRecent(workspaceId, kinds, limit)` read helper — replaces the §6.4 derivation as activity source | **Activity implements** (F9 swaps call site, no migration) |
| **All source modules** | `record(event, tx)` internal writer — called in-tx by workspace/members/projects/issues/comments/cycles services | **Activity implements** (call sites land per module here) |
| **Search (F10)** | Nothing — events are never searched (page filters are kind/actor only) | — (intentionally none) |
| **Notifications (F6)** | Nothing — the log never notifies; mentions/assignments keep their own fan-out | — (intentionally none) |

---

## 8. Migration workflow

Hand-modeled Prisma (like all prior features):

```bash
# 1 — add ActivityKind + ActivityEntityType enums + ActivityEvent model + back-relations
#     on Workspace/User (no back-relations on target tables — no FKs, D3)
# 2 — run
pnpm --filter @shipyard/api db:migrate -- --name add_activity_log
pnpm --filter @shipyard/api db:generate
```

- The migration produces: 1 table (`activity_event`), 2 enums, FKs + indexes above. No raw-SQL appends — Prisma expresses everything.
- Emission call sites land per source service in the same milestone (no separate migrations on source tables).
- The F1 Testcontainers harness applies migrations automatically each test run. No backfill job exists by design (spec Q6).

**Post-migration verification (manual, once):**

```sql
-- every row has a workspace and a frozen actor name
SELECT id FROM activity_event WHERE workspace_id IS NULL OR actor_name IS NULL OR actor_name = '';
-- every row has a rendered summary
SELECT id FROM activity_event WHERE summary IS NULL OR summary = '';
-- no orphan workspace refs (FK guarantees; sanity)
SELECT e.id FROM activity_event e LEFT JOIN workspace w ON w.id = e.workspace_id WHERE w.id IS NULL;
```

---

## 9. What we intentionally do NOT model

| Deferred | Why |
|---|---|
| Target FKs / back-relations from entity tables | Rejected in D3 — survival beats joinability for a historical record. |
| `updatedAt` / row mutation paths | Rejected — immutable rows (rule 3). |
| Read/unread state per user | Not notifications — the log is a browsed record, not an inbox. No per-user state. |
| Label/priority/date/resend micro-events | Spec §3.1 exclusion list — narrative, not exhaustiveness. |
| Comment-edit events | Excluded — edits already avoid notifications; same quiet applies here. |
| Export (CSV), retention jobs, TTL | Spec §6 out of scope. |
| Cross-workspace feed | Workspace-scoped like everything except notifications. |
| Realtime updates | Arch §11 — page loads on navigation; badge polling stays notifications-only. |
| Backfill of pre-feature history | Spec Q6 — starts empty, entity timelines cover the past. |

---

## 10. Open product questions — resolved at data layer

All eight spec §7 questions locked 2026-09-04: full taxonomy (§3.1/§2.1 enum); snapshots surviving deletion (D3/D5); frozen actor names (D4); page reads `/w/:slug/activity` 25/100 (api-design); uncapped retention (D6); no backfill (D6); strict in-tx emission (D2); any-member reads (api-design §4).

---

## 11. References

- Shipyard: `features/activity/spec.md`, `features/issues/data-model.md` (typed events D7, actor `SetNull` D3, cascade convention), `features/comments/data-model.md` (snapshot reasoning, same-tx side effects), `features/notifications/data-model.md` (internal emission D9, inverted cascades), `features/members/data-model.md` (membership vs user rows), `features/workspace/data-model.md` (identity, cascade), `features/auth/data-model.md` (identity), `features/dashboard/data-model.md` §7 (migration consumer), `00-architecture.md` §5/§8/§9, `ADR-001`, `ADR-002`, `Implementation Plan.md` F9
- Prisma indexes & referential actions: `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- Prisma cursor pagination: `https://www.prisma.io/docs/orm/prisma-client/queries/pagination#cursor-based-pagination`

---

*Next artifact: `api-design.md` — page endpoint (`GET` list with area/actor/entity filters, newest-first cursor), workspace-context guard chain (any member, readable-when-archived), error codes, per-area emission call-site wiring, and the dashboard panel migration.*
