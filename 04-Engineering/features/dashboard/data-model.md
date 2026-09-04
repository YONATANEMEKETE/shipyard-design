# Dashboard — Data Model

**Status:** Draft for review
**Last updated:** 2026-09-04
**Sources:** `features/dashboard/spec.md` · `features/issues/data-model.md` (F5 precedent — `issue`/`issue_history` read shapes, assignment addressability) · `features/projects/data-model.md` (F4 precedent — `project` card, derived progress, no audit table) · `features/cycles/data-model.md` (F7 precedent — active-cycle lookup, derived progress) · `features/comments/data-model.md` (F8 precedent — chronological reads, author cards) · `features/members/data-model.md` (F3 precedent — membership checks) · `00-architecture.md` §5, §8, §9 · `ADR-001` (Prisma + Postgres) · `ADR-002` (shared contracts) · `Implementation Plan.md` F9
**Owner:** `apps/api` — Prisma-owned (hand-modeled, like workspace/members/projects/issues/cycles/comments/notifications).

> **Locked scope (2026-09-04):** strict four panels (My Work / Current Cycle / Active Projects / Recent Activity) — no fifth attention aggregate; blocked/overdue ride as flags on My Work rows. Recent Activity in F9 is a lightweight derivation over `issue_history` + comment creations only. The full workspace **Activity Log** (workspace/member/project/issue/comment/cycle events, own page, all members can browse) is a separate upcoming feature with its own tables — F9 builds no activity infrastructure and migrates its panel onto that feed when it lands (§7).

---

## 1. Overview

Dashboard is a **view module**: a per-user, per-workspace composed landing view derived at read time from other features' tables. It stores no panel content, no snapshots, no counters — one new table exists only for the personal recently-viewed trail.

One new table, zero new enums:

| Table / Change | Purpose | Formalized by |
|---|---|---|
| `issue_view` | Personal recently-viewed trail: one row per (user, issue), bumped on revisit, pruned past 50 | **F9 (this milestone)** |

Everything else is read contracts over tables owned elsewhere (`issue`, `issue_history`, `comment`, `project`, `cycle`, `workspace_member`, `user`). No widened enums, no new columns on other tables.

---

## 2. Core schema (Prisma-owned)

### 2.1 `issue_view` — the personal trail (spec §3.2)

One row per (user, issue) pair. Revisiting bumps `viewedAt`; the trail is capped at 50 rows per (user, workspace) with oldest-first pruning; only the owner ever reads their rows (spec §3.2/rule 3, Q1 resolved personal-only).

| Column | Type | Attr | Notes |
|---|---|---|---|
| `userId` | `String` | FK → `user.id` `onDelete: Cascade` + `@@index([userId])` | Owner of the trail. User delete removes their trail (private data, same reasoning as `notification.recipientId Cascade`). |
| `issueId` | `String` | FK → `issue.id` `onDelete: Cascade` | The viewed issue. Issue delete removes its trail rows — no dead links (same reasoning as notification issue-cascade). |
| `workspaceId` | `String` | FK → `workspace.id` `onDelete: Cascade` | Denormalized scope: per-workspace cap/prune/read without joining `issue` (same convention as `issue_history.workspaceId`, `comment.workspaceId`). Workspace delete removes its trails. |
| `viewedAt` | `DateTime` | `@updatedAt`? No — `@default(now())`, set explicitly to `now()` on every bump | Recency position. Explicit writes (not `@updatedAt`) so creation and bump share one code path: upsert sets `viewedAt = now()`. |
| `createdAt` | `DateTime` | `@default(now())` | First-view time (display secondary; primary order is `viewedAt`). |

```prisma
model IssueView {
  userId      String
  issueId     String
  workspaceId String
  viewedAt    DateTime @default(now())
  createdAt   DateTime @default(now())

  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  issue     Issue     @relation(fields: [issueId], references: [id], onDelete: Cascade)
  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@id([userId, issueId])
  @@index([userId])
  @@index([userId, workspaceId, viewedAt])
  @@map("issue_view")
}
```

`@@id([userId, issueId])` makes "one entry per (user, issue)" (rule 3) structural — revisits upsert, never duplicate. `@@index([userId, workspaceId, viewedAt])` serves both the top-N trail read (`ORDER BY viewedAt DESC LIMIT 10`) and the prune (`DELETE` past 50 oldest) on one index.

`User` gains `issueViews IssueView[]`. `Issue` gains `views IssueView[]`. `Workspace` gains `issueViews IssueView[]`.

No raw-SQL indexes needed in F9 — Prisma expresses everything (no functional or partial indexes; the cap is a service-count rule, not a constraint).

### 2.2 Read contracts (no tables — derived panels)

Each panel is a bounded query over owning-module tables, reusing their card schemas verbatim (Implementation Plan: "reuse owning module query contracts instead of duplicating domain rules"):

| Panel | Source tables | Shape | Bound |
|---|---|---|---|
| My Work — assigned open | `issue WHERE workspaceId AND assigneeId = me AND archivedAt IS NULL AND status != DONE` | `issueCardSchema[]` | 10 |
| My Work — created open | `issue WHERE workspaceId AND creatorId = me AND archivedAt IS NULL AND status != DONE` | `issueCardSchema[]` | 10 |
| My Work — recently viewed | `issue_view WHERE userId = me AND workspaceId` ⨝ live `issue` rows, `ORDER viewedAt DESC` | `issueCardSchema[]` (+ `viewedAt`) | 10 of capped 50 |
| Current Cycle | `cycle WHERE workspaceId AND status = ACTIVE AND archivedAt IS NULL` (≤1 row, D6 guarantee) + progress derivation | `cycleCardSchema \| null` | 0–1 |
| Active Projects | `project WHERE workspaceId AND status = ACTIVE AND archivedAt IS NULL` + progress derivation | `projectCardSchema[]` | all (few per workspace; hard cap 20 for safety) |
| Recent Activity | `issue_history` (workspace scope) ∪ comment creations, `ORDER createdAt DESC` | `activityItemSchema[]` (heterogeneous, §4) | 20 |

"Open" = `status != DONE` **and** non-archived (locked): blocked issues are included (blocked is orthogonal work-to-do); completed and archived issues are not "work". Blocked/overdue surface as derived flags on the returned issue cards (`blocked`, `dueDate < today`) — no separate aggregates, no new queries (locked strict-four-panels).

The recently-viewed join keeps archived issues with their `archivedAt` flag (same tolerance as notifications) — the trail is a personal fact ("I looked at this"), and the landing page renders its own read-only banner. Deleted issues vanish via cascade and simply drop out of the join.

---

## 3. Key decisions & alternatives

### D1 — One table: `issue_view` only; panels are queries, not storage

**Decision:** the dashboard persists exactly the trail — everything else is computed per request from source tables. This is spec rule 2 made structural: there is literally nowhere for a snapshot to live, so drift is unrepresentable.

*Rejected:* a `dashboard_snapshot`/materialized panel cache — invalidation coupling to five modules' writes for a page that loads on navigation (spec §3.3: no auto-refresh required) and whose sources are already indexed. Revisit only with measured latency evidence, never preemptively.

### D2 — Trail identity is `@@id([userId, issueId])`, recency is `viewedAt`

**Decision:** composite PK enforces one-entry-per-pair; every view upserts `viewedAt = now()` (bump-to-top, rule 3). Cap 50 per (user, workspace) enforced in the recording tx: after upsert, `DELETE` rows past the 50 newest (single statement keyed off the §2.1 index).

*Rejected:* `@@unique([userId, workspaceId, issueId])` with a surrogate `id` — the workspace is functionally dependent on the issue, so the triple key adds nothing over the pair + denormalized scope column; the pair PK is the tighter invariant.

### D3 — Recording is a best-effort server side-effect on issue-detail read (locked)

**Decision:** the issues `GET detail` handler upserts the trail row in a fire-and-forget step that can never fail the read (try/catch around it; read returns regardless — rule 4). One code path means clients cannot forget to record, and the "revisits bump" semantic holds for every entry point (panel click, notification deep-link, search result, direct URL).

*Rejected:* explicit client `POST /recently-viewed` — doubles the request graph, lets-Terminal clients diverge, and turns a best-effort nicety into a second failure mode on the critical detail path.

### D4 — Recent Activity = `issue_history` ∪ comment creations, bound 20 (locked, reduced scope)

**Decision:** the F9 feed unions exactly two sources that already exist as event rows: `issue_history` (status/blocked/assignee/planning/archive events, with actor + old/new) and comment creations (`comment.createdAt`, author, issue ref). Project lifecycle and cycle transitions are **excluded** — F4 stores no event rows (deferred audit) and F7 stores no transition log, so "Sprint 13 started" is underivable without inventing facts. The exclusion is documented in the panel contract (empty causes stay truthful) rather than papered over.

*Rejected:* (a) deriving project/cycle events from `createdAt`/`updatedAt` — `updatedAt` cannot distinguish "started" from "renamed", so the feed would mislabel; (b) adding a dashboard-owned activity table written by project/cycle services now — that **is** the upcoming Activity Log feature, and smuggling its first half into F9 forks its design before planning. §7 records the migration path.

### D5 — Strict four panels; attention is flags, not aggregates (locked)

**Decision:** the composed payload carries exactly the spec §3.1 panels. Blocked/overdue needs surface as `blocked` + `dueDate` fields already on `issueCardSchema` — the web highlights them inside My Work ("blocked", "overdue" badges) with zero extra queries. The Plan's "blocked/overdue aggregates" wording is satisfied as derived presentation, not new endpoints.

### D6 — Current-cycle read trusts the D6 guarantee (≤1 row)

**Decision:** `WHERE status = ACTIVE AND archivedAt IS NULL` with `take: 1`-equivalent semantics; two rows is treated as a data-integrity bug (logged + Sentry, first row served) rather than a renderable state — the partial unique index underneath makes it unreachable except mid-migration. Empty ⇒ `currentCycle: null` plus the designed empty state (rule 5: empty states are data).

### D7 — Personal + workspace-scoped + permission-aware falls out of the sources (locked)

**Decision:** no dashboard-level permission logic exists. My Work filters by `me`; the trail filters by `me`; Current Cycle / Active Projects / Activity read workspace-visible entities every member may already list via their own endpoints. A member removed mid-session simply resolves no workspace context upstream (F2 guard) before dashboard code runs.

---

## 4. Shared contracts (`packages/shared`)

Added in F9, consumed by `api` and `web` (ADR-002). All entity cards are re-exported from their owning modules — never redefined.

```ts
// My Work groups — issueCardSchema reused verbatim (F5), plus trail recency where relevant
export const dashboardMyWorkSchema = z.object({
  assigned: z.array(issueCardSchema),   // open, assigned to me (≤10)
  created: z.array(issueCardSchema),    // open, created by me (≤10)
  recentlyViewed: z.array(              // live issues from my trail (≤10 of 50)
    issueCardSchema.extend({ viewedAt: z.string().datetime() }),
  ),
});

// Current cycle — cycleCardSchema reused verbatim (F7), nullable
export const dashboardCycleSchema = cycleCardSchema.nullable();

// Active projects — projectCardSchema reused verbatim (F4)
export const dashboardProjectsSchema = z.array(projectCardSchema);

// Recent activity — heterogeneous union over the two F9 sources (D4)
export const dashboardActivityKindSchema = z.enum([
  "ISSUE_STATUS_CHANGED",
  "ISSUE_BLOCKED_SET",
  "ISSUE_BLOCKED_CLEARED",
  "ISSUE_ASSIGNED",
  "ISSUE_UNASSIGNED",
  "ISSUE_PLANNING_CHANGED", // priority/project/cycle/due/title bucket (details in text)
  "ISSUE_ARCHIVED",
  "ISSUE_RESTORED",
  "ISSUE_CREATED",
  "COMMENT_CREATED",
]);

export const dashboardActivityItemSchema = z.object({
  kind: dashboardActivityKindSchema,
  actor: issueAssigneeCardSchema.nullable(), // "former member" fallback via SetNull legacy
  issue: z.object({
    id: z.string(),
    identifier: z.string(),
    title: z.string(),
  }),
  workspaceId: z.string(),
  commentId: z.string().nullable(), // set for COMMENT_CREATED (scroll target)
  text: z.string(),                 // server-rendered summary from old/new values (copy is data here, not client prose)
  createdAt: z.string().datetime(),
});

// Composed payload — one request, four panels (Implementation Plan: done-when #1)
export const dashboardSchema = z.object({
  workspaceId: z.string(),
  myWork: dashboardMyWorkSchema,
  currentCycle: dashboardCycleSchema,
  activeProjects: dashboardProjectsSchema,
  recentActivity: z.array(dashboardActivityItemSchema), // ≤20, newest first
});
```

Panel bounds (spec Q3 resolved): 10 / 10 / 10 / 1 / uncapped-projects / 20. The activity `text` is server-rendered from the source row's typed fields (e.g. `STATUS_CHANGED TODO→IN_PROGRESS`) — unlike notification copy, activity items have no stable client presenter yet, so the fact string ships with the item; the Activity Log feature may rationalize both later (§7).

---

## 5. Integrity invariants → spec rule mapping

| Spec rule | Enforcement point |
|---|---|
| 1 — personal + workspace-scoped, permission-aware | `me`-filters + `workspaceId` on every panel query; F2 workspace context upstream (D7) |
| 2 — derived at read time, no snapshots | D1: no panel storage exists; composed service issues bounded parallel reads per request |
| 3 — one trail entry per (user, issue); bump; cap 50 prune-oldest | `@@id([userId, issueId])` + upsert-bump + in-tx prune (§6.1) |
| 4 — recording never fails the read | D3: best-effort side effect, isolated from the detail response path |
| 5 — empty states are data | Nullable/empty-able payload fields (`currentCycle: null`, `[]` groups); no 404-for-empty anywhere |
| 6 — activity bounded, newest-first | `(createdAt DESC, id DESC)` union capped at 20 (§6.4) |

Integrity summary — constraints added in F9:

| Constraint | Where | Purpose |
|---|---|---|
| `@@id([userId, issueId])` | `issue_view` | One trail entry per pair (rule 3) |
| FK `issue_view.userId → user` `Cascade` | `issue_view` | Trail dies with the account (private data) |
| FK `issue_view.issueId → issue` `Cascade` | `issue_view` | No dead trail links |
| FK `issue_view.workspaceId → workspace` `Cascade` | `issue_view` | Dies with its workspace |
| `@@index([userId, workspaceId, viewedAt])` | `issue_view` | Top-N trail read + prune on one index |

---

## 6. Lifecycle semantics at the data layer

The dashboard has no lifecycle of its own — one recording write plus five bounded read shapes, all enforcing the "no N+1" done-criterion via fixed fan-out (parallel queries, each capped; per-row follow-ups forbidden).

### 6.1 Trail recording (the only write, spec §3.2)

```text
issues GET detail handler, after the 200 card is built:
  best-effort (never throws into the response):
    tx {
      UPSERT issue_view (userId, issueId) SET workspaceId, viewedAt = now()
      DELETE FROM issue_view WHERE userId AND workspaceId
        AND (userId, issueId) NOT IN (SELECT ... ORDER BY viewedAt DESC LIMIT 50)
    }
```

Records for every detail view regardless of entry point (D3). Archived-issue views record normally (the trail is personal history, not issue mutation — the D9 freeze covers comment writes, not "I looked"). Non-member/unknown-issue reads never reach the recorder (F2/F5 guards fire first). Prune cost is one indexed statement per view — negligible against the detail read it follows.

### 6.2 My Work reads

Three parallel capped queries, all `AND archivedAt IS NULL AND status != DONE` ("open", locked §2):

```text
assigned: WHERE workspaceId AND assigneeId = me      ORDER updatedAt DESC LIMIT 10
created:  WHERE workspaceId AND creatorId = me       ORDER updatedAt DESC LIMIT 10
recent:   issue_view WHERE userId = me AND workspaceId ORDER viewedAt DESC LIMIT 10
          ⨝ issue rows (inner join — cascaded deletions drop out naturally)
```

Blocked/overdue arrive as row flags for client badges (D5) — no extra statements.

### 6.3 Current cycle + active projects reads

```text
currentCycle: WHERE workspaceId AND status = ACTIVE AND archivedAt IS NULL  (≤1, D6)
              + progress derivation (two COUNTs, F7 D8 shape)
activeProjects: WHERE workspaceId AND status = ACTIVE AND archivedAt IS NULL
              ORDER targetDate/startDate per F4 list convention  (hard cap 20)
              + progress derivation per project (two COUNTs each — bounded by the cap, still flat queries, no N+1)
```

Progress counts reuse the owning modules' derivation queries (never stored, never recomputed here).

### 6.4 Recent activity read (lightweight, D4)

```text
history leg:  issue_history WHERE workspaceId ORDER (createdAt DESC, id DESC) LIMIT 20
comments leg: comment ⨝ issue WHERE comment.workspaceId ORDER (createdAt DESC, id DESC) LIMIT 20
merge in service → sort DESC → take 20 → map to dashboardActivityItemSchema
```

Two flat queries, service-side merge — no per-item actor fetch (actor cards batch-loaded in one `WHERE id IN (...)` where needed, or joined). Project/cycle lifecycle rows do not exist to query; the feed never claims otherwise (§4 `dashboardActivityKindSchema` has no project/cycle kinds — the Activity Log feature adds them, §7).

### 6.5 Archived-workspace interaction

`workspace.status = ARCHIVED` leaves the dashboard readable (frozen workspaces stay browsable, F2): all four panels serve normally; the trail recorder keeps working (personal history, not container mutation); no new issues/cycles/projects can arise upstream to change the panels. No `rejectArchived` guard on the dashboard route — it is a `GET` with zero container mutations.

---

## 7. Forward handoffs — what this model does NOT contain

| Consumer / Successor | Contract F9 provides / leaves open | Landed |
|---|---|---|
| **Activity Log (upcoming feature)** | F9 builds NO activity infrastructure (D4): no event table, no subscription, no backfill. The Log will own workspace-scoped event rows (workspace/member/project/issue/comment/cycle events) with its own page; on landing it **replaces** the §6.4 derivation as the Recent Activity source (kinds extend, panel bound stays 20). F9's `dashboardActivityItemSchema` is shaped to extend, not to be thrown away. | **Upcoming implements** |
| **Issues (F5)** | Recording call site lives in the issues detail-read path (§6.1) — F9 documents it; F5's table is untouched (no new issue columns). | **F9 implements** (one hook, no migration on `issue`) |
| **Search (F10)** | Nothing — dashboard panels are not searchable surfaces; trail rows are never searched. | — (intentionally none) |
| **Settings (F11)** | Nothing — no dashboard preferences in MVP (spec §6: no customizable layouts/widgets). | — (intentionally none) |
| **Notifications (F6)** | Nothing — badge polling stays independent (§5 of notifications data-model); dashboard never reads `notification`. | — (intentionally none) |

---

## 8. Migration workflow

Hand-modeled Prisma (like workspace/members/projects/issues/cycles/comments/notifications):

```bash
# 1 — add IssueView model + back-relations on User/Issue/Workspace
# 2 — run
pnpm --filter @shipyard/api db:migrate -- --name add_dashboard_issue_view
pnpm --filter @shipyard/api db:generate
```

- The migration produces: 1 table (`issue_view`), 0 new enums, FKs + indexes above. No raw-SQL appends — Prisma expresses everything F9 needs.
- No backfill: the trail starts empty for every user; first detail views populate it (empty-trail state is designed, rule 5).
- The F1 Testcontainers harness applies migrations automatically each test run.

**Post-migration verification (manual, once):**

```sql
-- one entry per (user, issue) (PK guarantees; confirms no legacy drift)
SELECT user_id, issue_id, count(*) FROM issue_view GROUP BY 1,2 HAVING count(*)>1;
-- every trail row's workspace matches its issue's workspace
SELECT v.user_id, v.issue_id FROM issue_view v JOIN issue i ON i.id = v.issue_id
  WHERE v.workspace_id != i.workspace_id;
-- cap holds per (user, workspace) after soak (investigate only if violated — service prunes)
SELECT user_id, workspace_id, count(*) FROM issue_view GROUP BY 1,2 HAVING count(*)>50;
```

---

## 9. What we intentionally do NOT model

| Deferred | Why |
|---|---|
| Workspace activity event table / subscriptions / backfill | The upcoming Activity Log feature owns all of it (D4, §7) — F9 must not fork its design early. |
| Project/cycle lifecycle kinds in Recent Activity | Underivable without event rows (D4) — arrive with the Activity Log. |
| Panel snapshots / materialized cache | Rejected in D1 — derived per request, no drift surface. |
| Customizable layouts, widgets, pinning, reorder | Spec §6 out of scope. |
| Analytics, reports, workload charts | Spec §6 out of scope. |
| Cross-workspace dashboard | Spec §6 out of scope — strictly one workspace per payload. |
| Dashboard preferences (panel order, hidden panels) | Spec §6 + Settings-owns-preferences — no F9 preference rows. |
| `updatedAt` on `issue_view` | Rejected in §2.1 — `viewedAt`/`createdAt` carry the semantics; a third timestamp is noise. |
| Per-panel pagination | Panels are fixed-bound glimpses with drill-down links, not paginated lists — full browsing lives on the owning pages. |

---

## 10. Open product questions — resolved at data layer

| Spec §7 | Decision |
|---|---|
| 1 — recently viewed privacy | **Locked personal-only:** `userId`-scoped reads, no workspace-visible variant; leavers' trails stay private to them (D7, §6.1). |
| 2 — activity feed scope | **Locked workspace-wide:** all members' issue/comment events, newest-first (D7, §6.4). Project/cycle kinds arrive with the Activity Log (§7). |
| 3 — panel item counts | **Locked:** 10 assigned / 10 created / 10 recent (of capped 50) / 0–1 cycle / active projects uncapped (cap 20 guard) / 20 activity (§2.2, §4). |

---

## 11. References

- Shipyard: `features/dashboard/spec.md`, `features/issues/data-model.md` (read shapes, history pattern, cascade convention), `features/projects/data-model.md` (cards, derived progress, no-audit precedent), `features/cycles/data-model.md` (active lookup D6, derived progress D8), `features/comments/data-model.md` (chronological reads, author cards, join conventions), `features/members/data-model.md` (membership checks, index conventions), `features/workspace/data-model.md` (identity, cascade), `00-architecture.md` §5/§8/§9, `ADR-001`, `ADR-002`, `Implementation Plan.md` F9
- Prisma indexes & referential actions: `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`

---

*Next artifact: `api-design.md` — single composed `GET` endpoint contract (four panels, bounds, empty-state shapes), workspace-context guard chain (readable-when-archived, no `rejectArchived`), error codes, the issue-detail recording hook, and the Activity Log migration note.*
