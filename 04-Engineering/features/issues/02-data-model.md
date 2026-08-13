# Issues — Data Model

**Module:** `apps/api/src/modules/issues`
**Status:** Draft v0.1 — 2026-08-12
**Stack:** Prisma + PostgreSQL · tsvector full-text (ADR-001)
**PRD source:** §5.5 Issues

---

## 1. Overview

Issues owns **three tables** — `Issue`, `Label`, and the `IssueLabel` join — plus the tsvector search index that the search module queries.

| Table | Purpose |
|---|---|
| `Issue` | The unit of work + all planning fields |
| `Label` | Workspace-scoped tags (unique per workspace) |
| `IssueLabel` | Many-to-many join (issue ⇄ label) |

---

## 2. Prisma Schema

```prisma
// ============ ISSUES MODULE ============

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
  id           String        @id @default(cuid())
  workspaceId  String
  identifier   String        // "SHIP-024" — unique per workspace, never reused
  title        String        // required, 1–200 chars
  description  String?       // plain text (rich text is post-MVP)
  status       IssueStatus   @default(BACKLOG)
  priority     IssuePriority @default(NO_PRIORITY)
  projectId    String?       // SetNull on project delete (PRD atomic unassignment)
  cycleId      String?       // SetNull on cycle delete
  assigneeId   String?       // SetNull if user ever deleted (future)
  creatorId    String        // immutable
  dueDate      DateTime?     // date-only semantics (store UTC midnight)
  isBlocked    Boolean       @default(false)
  blockedReason String?
  archivedAt   DateTime?     // null = active; archival pattern
  createdAt    DateTime      @default(now())
  updatedAt    DateTime      @updatedAt

  // search vector — generated column, kept in sync by the DB
  searchVector Unsupported("tsvector")? // see migration SQL below

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  project   Project?  @relation(fields: [projectId], references: [id], onDelete: SetNull)
  cycle     Cycle?    @relation(fields: [cycleId], references: [id], onDelete: SetNull)
  assignee  User?     @relation("AssignedIssues", fields: [assigneeId], references: [id], onDelete: SetNull)
  creator   User      @relation("CreatedIssues", fields: [creatorId], references: [id])
  labels    IssueLabel[]
  comments  Comment[] // cascade via comments module
  activities IssueActivity[]

  @@unique([workspaceId, identifier])
  @@index([workspaceId, status])
  @@index([workspaceId, assigneeId])
  @@index([workspaceId, projectId])
  @@index([workspaceId, cycleId])
  @@index([workspaceId, priority])
  @@index([workspaceId, isBlocked])
  @@index([workspaceId, dueDate])
  @@index([workspaceId, createdAt])
}

model Label {
  id          String   @id @default(cuid())
  workspaceId String
  name        String   // trimmed, case-insensitive unique per workspace
  color       String   @default("#B45309") // ds-brand default; UI palette picker
  createdAt   DateTime @default(now())

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  issues    IssueLabel[]

  @@unique([workspaceId, name])
}

model IssueLabel {
  issueId String
  labelId String
  issue   Issue @relation(fields: [issueId], references: [id], onDelete: Cascade)
  label   Label @relation(fields: [labelId], references: [id], onDelete: Cascade)

  @@id([issueId, labelId])
  @@index([labelId])
}

// ============ ISSUE ACTIVITY (history — feeds issue detail + dashboard feed) ============

model IssueActivity {
  id        String   @id @default(cuid())
  issueId   String
  actorId   String   // who performed the change
  action    String   // "STATUS_CHANGED" | "BLOCKED" | "UNBLOCKED" | "ASSIGNED" |
                     // "PLANNING_CHANGED" | "CREATED" | "ARCHIVED" | "RESTORED"
  detail    Json?    // { from, to } for status/priority/assignee changes
  createdAt DateTime @default(now())

  issue Issue @relation(fields: [issueId], references: [id], onDelete: Cascade)

  @@index([issueId, createdAt])
}
```

**Migration-time SQL** (Prisma cannot express generated columns):

```sql
ALTER TABLE "Issue" ADD COLUMN "searchVector" tsvector
  GENERATED ALWAYS AS (
    to_tsvector('english', coalesce(title,'') || ' ' || coalesce(description,''))
  ) STORED;

CREATE INDEX issue_search_idx ON "Issue" USING GIN ("searchVector");
```

---

## 3. Field Notes & Design Rationale

- **`identifier` + per-workspace sequence** — allocated inside the create transaction: read `Workspace.lastIssueNumber`, increment, write both. The `@@unique([workspaceId, identifier])` constraint is the backstop; a P2002 there means a retry with the next number (never reuse).
- **`searchVector` generated column** — the DB maintains it; no app code can desync it. GIN index for `@@` queries (ADR-001: Postgres FTS). Search *orchestration* lives in the search module; the column is owned here.
- **`onDelete: SetNull` on project/cycle/assignee** — PRD atomicity for free: project deletion unassigns issues in the same DB transaction; cycle deletion likewise; no app-level choreography.
- **`isBlocked` + `blockedReason`** — the orthogonal flag; `DONE` clearing is service logic (domain rule 6), enforced in one place.
- **`archivedAt` pattern** — status/blocked state are *not* snapshotted: archive only stamps the timestamp, restore clears it, and the pre-archive state is naturally preserved (domain model §2.1).
- **`dueDate` as UTC midnight** — date-only semantics; no timezone drift in comparisons.
- **`IssueActivity.detail` as Json** — `{ from, to }` pairs; flexible without schema churn (statuses, priorities, assignees, blocked). Actions are an enum string for cheap filtering (dashboard feed).
- **Labels:** `@@unique([workspaceId, name])` — Prisma's case-sensitive unique; the *normalized* (trim + lowercase) check runs in the service (same pattern as projects/cycles). Color defaults to `ds-brand`; palette picker in UI.

---

## 4. Indexes & Constraints Summary

| Object | Type | Why |
|---|---|---|
| `Issue(workspaceId, identifier)` | UNIQUE | Display-id uniqueness per workspace |
| `Issue(workspaceId, status)` | INDEX | Kanban columns + status filters |
| `Issue(workspaceId, assigneeId)` | INDEX | My Issues + assignee filter |
| `Issue(workspaceId, projectId)` | INDEX | Project issue list |
| `Issue(workspaceId, cycleId)` | INDEX | Cycle issue list |
| `Issue(workspaceId, priority)` / `(workspaceId, isBlocked)` / `(workspaceId, dueDate)` | INDEX | Filter hot paths |
| `Issue(workspaceId, createdAt)` | INDEX | Default sort + cursor pagination |
| `Issue.searchVector` | GIN | Full-text search (`@@`) |
| `Label(workspaceId, name)` | UNIQUE | Label identity |
| `IssueActivity(issueId, createdAt)` | INDEX | History + feed |

---

## 5. Data Lifecycle

| Event | SQL-level behavior |
|---|---|
| Create | TRANSACTION: bump `Workspace.lastIssueNumber` → INSERT `Issue` (identifier = SHIP-N, defaults) → [if assignee set] notification insert via notifications service |
| Update planning fields | UPDATE + INSERT `IssueActivity` (PLANNING_CHANGED / ASSIGNED) + notification on assignee change — one transaction |
| Status change (dropdown OR drag) | UPDATE status + INSERT `IssueActivity` (STATUS_CHANGED {from,to}) + auto-clear blocked on DONE |
| Toggle blocked | UPDATE `isBlocked`/`blockedReason` + INSERT activity (BLOCKED/UNBLOCKED) |
| Archive / Restore | UPDATE `archivedAt` + INSERT activity |
| Delete | DELETE `Issue` → comments + activities cascade; notifications cascade (dead references must not survive) |
| Project deleted | `Issue.projectId` auto-SetNull (same DB txn) — issues stay, unassigned |
| Cycle deleted | `Issue.cycleId` auto-SetNull — issues stay, unassigned |
| Workspace deleted | Cascade (workspace contract) |

---

## 6. Sizing & Free-Tier Fit

The dominant table of the product: ~1KB per issue row + activities (~200B each) + index overhead. 10k issues + 50k activities ≈ 25–40MB — comfortably inside Neon's 0.5GB free tier; the GIN index is the largest single object and stays well under limits at this scale. Cursor pagination (open question 4) keeps list queries bounded regardless.

---

## 7. Decisions Adopted (from domain model open questions)

| # | Question | Decision |
|---|---|---|
| 1 | Priority scale | **NO_PRIORITY · URGENT · HIGH · MEDIUM · LOW** (Linear-style) |
| 2 | Label creation | **Any member** can create labels (workspace-scoped, service-normalized uniqueness) |
| 3 | Identifier | **`SHIP-###`** — constant prefix + per-workspace sequence (design sample); prefix becomes configurable later |
| 4 | Pagination | **Cursor pagination** on issue lists (`cursor` + `limit`, default 50) |
