# Cycles — Data Model

**Module:** `apps/api/src/modules/cycles`
**Status:** Draft v0.1 — 2026-08-12
**Stack:** Prisma + PostgreSQL — with **DB-level scheduling enforcement** (exclusion constraint + partial unique index)
**PRD source:** §5.7 Cycles

---

## 1. Overview

Cycles owns **two tables** — `Cycle` and `CycleActivity` — plus the two Postgres constructs that make the scheduling contract race-proof at the database level.

| Table | Purpose |
|---|---|
| `Cycle` | The iteration + dates + lifecycle state |
| `CycleActivity` | Lifecycle history (started/completed/reopened/archived/restored/deleted) |

---

## 2. Prisma Schema

```prisma
// ============ CYCLES MODULE ============

enum CycleStatus {
  PLANNED
  ACTIVE
  COMPLETED
}

model Cycle {
  id             String      @id @default(cuid())
  workspaceId    String
  name           String      // display name as typed
  nameNormalized String      // trimmed + lowercased — uniqueness key
  goal           String?
  startDate      DateTime    // required — UTC midnight (date-only semantics)
  endDate        DateTime    // required — inclusive
  status         CycleStatus @default(PLANNED)
  archivedAt     DateTime?   // null = non-archived; restore target = status
  createdAt      DateTime    @default(now())
  updatedAt      DateTime    @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  issues    Issue[]   // Issue.cycleId onDelete: SetNull (defined in issues model)
  activities CycleActivity[]

  @@unique([workspaceId, nameNormalized])
  @@index([workspaceId, status])
  @@index([workspaceId, startDate, endDate])
}

model CycleActivity {
  id        String   @id @default(cuid())
  cycleId   String
  actorId   String
  action    String   // "CREATED" | "UPDATED" | "STARTED" | "COMPLETED" |
                     // "REOPENED" | "ARCHIVED" | "RESTORED" | "DELETED"
  detail    Json?    // { from, to } where meaningful (status, dates)
  createdAt DateTime @default(now())

  cycle Cycle @relation(fields: [cycleId], references: [id], onDelete: Cascade)

  @@index([cycleId, createdAt])
}
```

**Migration-time SQL — the scheduling enforcement (Prisma cannot express these):**

```sql
-- AT MOST ONE ACTIVE CYCLE per workspace (same pattern as single-owner):
CREATE UNIQUE INDEX cycle_single_active
  ON "Cycle" (workspace_id) WHERE status = 'ACTIVE';

-- NON-OVERLAPPING date ranges for non-archived cycles (inclusive bounds):
CREATE EXTENSION IF NOT EXISTS btree_gist;  -- required for the WHERE clause

ALTER TABLE "Cycle" ADD CONSTRAINT cycle_no_overlap
  EXCLUDE USING gist (
    workspace_id WITH =,
    daterange(start_date::date, end_date::date, '[]') WITH &&
  )
  WHERE (status <> 'ARCHIVED' AND archived_at IS NULL);
```

---

## 3. Field Notes & Design Rationale

- **`nameNormalized`** — same pattern as projects: DB-enforced case-insensitive uniqueness (`@@unique([workspaceId, nameNormalized])`).
- **`startDate`/`endDate` as UTC-midnight DateTime** — date-only semantics; the exclusion constraint casts to `daterange` with **inclusive `'[]'` bounds** (PRD: dates are inclusive — a following cycle starts *after* the preceding end).
- **`status` + `archivedAt`** — archived cycles carry their stored pre-archive status in the status column (restore target); `archivedAt` marks the lifecycle state. The exclusion constraint's `WHERE` excludes archived rows so **history never blocks new scheduling** — but restore re-enters the constraint (guarded at service level too).
- **`CycleActivity`** — lifecycle events only; issue add/remove to cycles is recorded by the *issues* module's activity (the cycleId field lives there).
- **No progress column** — derived from issues on read (COUNT by status), identical pattern to projects.

---

## 4. DB-Enforced vs Service-Enforced Invariants

| Invariant | Enforced by |
|---|---|
| One active cycle per workspace | **DB** (partial unique index `cycle_single_active`) |
| Non-overlapping non-archived ranges | **DB** (exclusion constraint `cycle_no_overlap`) |
| Name uniqueness (normalized) | **DB** (unique + nameNormalized) |
| Transition legality (start/complete/reopen/…) | **Service** (state machine — the DB can't express it) |
| Restore/reopen no-overlap re-check | **Service** (+ DB constraint as referee) |
| Delete only future Planned cycles | **Service** |
| Completed = read-only | **Service** (+ data-layer write guard) |

**Why both layers:** the service produces friendly errors (`CYCLE_OVERLAP`, `CYCLE_ACTIVE_LIMIT`); the DB guarantees correctness even when two users start/overlap simultaneously. A constraint violation (Prisma P2004/P2002) maps to the same friendly codes.

---

## 5. Data Lifecycle

| Event | SQL-level behavior |
|---|---|
| Create | INSERT `Cycle` (PLANNED) + INSERT activity (CREATED) — constraint guards overlap |
| Update (Planned/Active) | UPDATE name/goal/dates + activity (UPDATED) — date edits re-check the constraint |
| Start | UPDATE `status = ACTIVE` + activity (STARTED) — partial unique index rejects a second active |
| Complete | UPDATE `status = COMPLETED` + activity (COMPLETED) — issues untouched |
| Reopen | UPDATE `status = ACTIVE` + activity (REOPENED) — active-limit + overlap re-checked |
| Archive | UPDATE `archivedAt = now` (status preserved) + activity (ARCHIVED) — leaves the constraint scope |
| Restore | UPDATE `archivedAt = null` + activity (RESTORED) — re-enters constraint scope (overlap must be clear) |
| Delete (future Planned) | DELETE `Cycle` → `Issue.cycleId` auto-SetNull (same statement) + activities cascade + name released |
| Workspace deleted | Cascade |

---

## 6. Sizing & Free-Tier Fit

Cycle rows ~400 bytes; activity ~200 bytes. Hundreds of cycles + history ≈ 1MB — trivial. The GiST exclusion index is small at this scale; `btree_gist` adds negligible overhead.

---

## 7. Decisions Adopted (from domain model open questions)

| # | Question | Decision |
|---|---|---|
| 1 | Cycle length guardrails | **None** — any valid non-overlapping range |
| 2 | Start date in the past | **Allowed** (cycles can start late) |
| 3 | Start conflict UX | `CYCLE_ACTIVE_LIMIT` 409 — web shows the conflicting active cycle |
