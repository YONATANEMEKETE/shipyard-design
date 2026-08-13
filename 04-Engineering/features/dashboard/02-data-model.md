# Dashboard — Data Model

**Module:** `apps/api/src/modules/dashboard`
**Status:** Draft v0.1 — 2026-08-12
**Stack:** Prisma + PostgreSQL
**PRD source:** §5.4 Dashboard

---

## 1. Overview

Dashboard owns **one table** — `RecentlyViewed` — the only state it needs. Every panel is composed from the owning modules' tables at read time (no snapshots, no dashboard cache).

| Table | Purpose |
|---|---|
| `RecentlyViewed` | Per-user trail of viewed issues (My Work panel + future recents) |

---

## 2. Prisma Schema

```prisma
// ============ DASHBOARD MODULE ============

model RecentlyViewed {
  id        String   @id @default(cuid())
  userId    String
  issueId   String
  viewedAt  DateTime @default(now())

  user  User  @relation("RecentlyViewedIssues", fields: [userId], references: [id], onDelete: Cascade)
  issue Issue @relation(fields: [issueId], references: [id], onDelete: Cascade)

  @@unique([userId, issueId])   // one row per issue per user — revisit = bump
  @@index([userId, viewedAt])   // "recently viewed, newest first" + prune order
}
```

---

## 3. Field Notes & Design Rationale

- **Upsert semantics** — `@@unique([userId, issueId])` + `viewedAt = now` on conflict: revisits refresh recency without growing the table.
- **Cap 50 per user** — enforced in the same upsert path: after insert, `DELETE WHERE userId = $1 AND id NOT IN (SELECT id ... ORDER BY viewedAt DESC LIMIT 50)` — a single bounded statement, oldest pruned. (Prune runs at most once per new unique view, not on every revisit.)
- **Cascade on issue delete** — a deleted issue vanishes from recents automatically (consistent with notifications).
- **`viewedAt` index** — serves both the newest-first read and the prune order.
- **No dashboard settings table** — view preferences (list/kanban) already live in the settings module; custom layouts are post-MVP.

---

## 4. Indexes & Constraints Summary

| Object | Type | Why |
|---|---|---|
| `RecentlyViewed(userId, issueId)` | UNIQUE | Upsert key (one row per issue per user) |
| `RecentlyViewed(userId, viewedAt)` | INDEX | Newest-first reads + pruning |

---

## 5. Data Lifecycle

| Event | SQL-level behavior |
|---|---|
| Issue detail read | Best-effort upsert (userId, issueId, viewedAt = now) + prune-to-50 — **never fails the read** (failure logged only) |
| Revisit same issue | Bump `viewedAt` (same upsert) — no growth |
| Issue deleted | Row cascades away |
| User deleted | Rows cascade away |
| Workspace deleted | Cascade via issue |

**Everything else is read-time composition:**

| Panel | Query (all scoped by workspaceId) |
|---|---|
| Assigned (My Work) | `Issue WHERE assigneeId = me AND archivedAt IS NULL AND status != DONE` — limit 10, newest updated first |
| Created (My Work) | `Issue WHERE creatorId = me AND archivedAt IS NULL AND status != DONE` — limit 10 |
| Recently viewed | `RecentlyViewed JOIN Issue WHERE userId = me AND issue NOT archived/deleted` — limit 10, viewedAt desc |
| Current Cycle | Active cycle + `COUNT(issues)` / `COUNT(status = DONE)` progress — null-safe |
| Active Projects | `Project WHERE status = ACTIVE AND archivedAt IS NULL` — with per-project progress (one grouped query) — limit 10 |
| Recent Activity | UNION of `IssueActivity` + `ProjectActivity` + `Comment` (createdAt) — limit 20, newest first |

All panel queries run in **parallel** (Promise.all) — the composed request is one HTTP call with ~6 bounded queries.

---

## 6. Sizing & Free-Tier Fit

RecentlyViewed ≈ 50 rows/user × 1k users = 50k rows ≈ 10MB — trivial. Activity aggregation is bounded by the LIMIT clauses; no index additions beyond those above.

---

## 7. Decisions Adopted (from domain model open questions)

| # | Question | Decision |
|---|---|---|
| 1 | Recently viewed privacy | **Personal** — only the owner sees it |
| 2 | Activity feed scope | **Workspace-wide** (all members' events, newest first) |
| 3 | Panel item counts | **10** per My Work group · **10** active projects · **20** activity events |
