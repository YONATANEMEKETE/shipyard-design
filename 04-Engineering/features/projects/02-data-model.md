# Projects — Data Model

**Module:** `apps/api/src/modules/projects`
**Status:** Draft v0.1 — 2026-08-12
**Stack:** Prisma + PostgreSQL
**PRD source:** §5.6 Projects

---

## 1. Overview

Projects owns **two tables** — `Project` and `ProjectActivity` (status/ownership/archive history). Progress is **derived** — no stored column (computed from issue statuses on read).

| Table | Purpose |
|---|---|
| `Project` | The initiative + planning fields + lifecycle state |
| `ProjectActivity` | History: status changes, ownership transfers, archive/restore |

---

## 2. Prisma Schema

```prisma
// ============ PROJECTS MODULE ============

enum ProjectStatus {
  PLANNED
  ACTIVE
  COMPLETED
}

model Project {
  id              String        @id @default(cuid())
  workspaceId     String
  name            String        // display name as typed
  nameNormalized  String        // trimmed + lowercased — uniqueness key (open Q3)
  description     String?
  status          ProjectStatus @default(PLANNED)
  ownerId         String        // Project Owner — grants no permissions
  startDate       DateTime?     // UTC midnight (date-only semantics)
  targetDate      DateTime?
  archivedAt      DateTime?     // null = non-archived; lifecycle state
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  owner     User      @relation("OwnedProjects", fields: [ownerId], references: [id])
  issues    Issue[]   // Issue.projectId onDelete: SetNull (defined in issues model)
  activities ProjectActivity[]

  @@unique([workspaceId, nameNormalized])
  @@index([workspaceId, status])
  @@index([workspaceId, ownerId])
  @@index([workspaceId, archivedAt])
}

model ProjectActivity {
  id        String   @id @default(cuid())
  projectId String
  actorId   String   // who performed the change
  action    String   // "CREATED" | "STATUS_CHANGED" | "OWNER_CHANGED" |
                     // "ARCHIVED" | "RESTORED" | "DELETED"
  detail    Json?    // { from, to } for status/owner changes
  createdAt DateTime @default(now())

  project Project @relation(fields: [projectId], references: [id], onDelete: Cascade)

  @@index([projectId, createdAt])
}
```

---

## 3. Field Notes & Design Rationale

- **`nameNormalized`** — the DB-level enforcement of the domain's case-insensitive, trimmed uniqueness (open question 3 adopted): the service writes `name.trim().toLowerCase()` into this column; `@@unique([workspaceId, nameNormalized])` makes the invariant race-proof. The display `name` keeps the user's original casing.
- **No stored progress** — derived on read: `COUNT(issues WHERE status = DONE) / COUNT(issues)` per project (one grouped query for lists). Storing it would invite drift; the issue table is the single source of truth. (If list queries ever need it hot, a materialized view is the documented upgrade — not MVP.)
- **`status` enum + `archivedAt`** — the standard Shipyard split: operational status switches freely; Archived is a lifecycle state with a timestamp, excluded from the status control.
- **`ownerId`** — exactly one Project Owner; `onDelete` for users is **SetNull-averse**: ownership must never be null. Since user-account deletion is future work, the FK is left as-is and the *auto-transfer* path (members contract) guarantees a member leaving never leaves an owned project behind. When user deletion lands, the transfer rule covers it.
- **Dates as UTC midnight** — date-only semantics, consistent with issues.
- **`ProjectActivity.detail` Json** — same pattern as `IssueActivity`; action enum strings keep the dashboard feed cheap.

---

## 4. Indexes & Constraints Summary

| Object | Type | Why |
|---|---|---|
| `Project(workspaceId, nameNormalized)` | UNIQUE | Case-insensitive name uniqueness, DB-enforced |
| `Project(workspaceId, status)` | INDEX | List sections + Kanban columns + status filter |
| `Project(workspaceId, ownerId)` | INDEX | "Projects owned by X" (transfer lists, member removal) |
| `Project(workspaceId, archivedAt)` | INDEX | Archived list |
| `ProjectActivity(projectId, createdAt)` | INDEX | Activity feed |

**No tsvector here** — search over projects reuses the shared search orchestration; if project search needs FTS, the same generated-column pattern applies (decide in the search module).

---

## 5. Data Lifecycle

| Event | SQL-level behavior |
|---|---|
| Create | INSERT `Project` (name + nameNormalized; creator as owner; status PLANNED) + INSERT `ProjectActivity` (CREATED) — one transaction |
| Update (name/description/dates) | UPDATE + activity row if status/owner changed |
| Status change (control or drag) | UPDATE `status` + INSERT `ProjectActivity` (STATUS_CHANGED {from,to}) — same endpoint, no confirmation |
| Transfer ownership | TRANSACTION: UPDATE `ownerId` → target (re-verified member inside txn) + INSERT activity (OWNER_CHANGED) — recipient's workspace role untouched |
| Archive | UPDATE `archivedAt = now` + activity (ARCHIVED) — status column untouched (restore target) |
| Restore | UPDATE `archivedAt = null` + activity (RESTORED) — returns to stored status |
| Delete | DELETE `Project` → **`Issue.projectId` auto-SetNull in the same DB transaction** + activities cascade + name released (row gone) |
| Member removed/leaves | UPDATE `ownerId → workspaceOwnerId` (called by members inside its transaction) + activity (OWNER_CHANGED) |
| Workspace deleted | Cascade |

**Failure semantics:** delete + unassignment are one DB statement — atomic by definition; a failed transfer leaves ownership unchanged.

---

## 6. Sizing & Free-Tier Fit

Project rows ~500 bytes; activity rows ~200 bytes. 1k projects + 10k activity rows ≈ 3MB — trivial within Neon's free tier. No pagination needed for project lists in MVP (open question 2 adopted: revisit if a workspace exceeds ~200 projects).

---

## 7. Decisions Adopted (from domain model open questions)

| # | Question | Decision |
|---|---|---|
| 1 | Activity granularity | Same model as issues (action enum + Json detail) — feeds the project activity section |
| 2 | List pagination | **None in MVP** — projects are bounded; revisit at scale |
| 3 | Name normalization | **`nameNormalized` column** + `@@unique(workspaceId, nameNormalized)` — DB-enforced case-insensitive uniqueness |
