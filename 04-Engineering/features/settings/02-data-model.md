# Settings — Data Model

**Module:** `apps/api/src/modules/settings`
**Status:** Draft v0.1 — 2026-08-12
**Stack:** Prisma + PostgreSQL
**PRD source:** §5.11 Settings

---

## 1. Overview

Settings owns **one new table** — `ViewPreference`. Profile and theme fields live on `User` (auth's table, written through the settings service); avatars live in R2 with only a signed URL in the DB.

| Table / field | Owner | Written by |
|---|---|---|
| `User.name` / `User.image` / `User.theme` | auth data model | settings service |
| `ViewPreference` | settings ✅ | settings service |
| Avatar object | R2 bucket (ADR-004) | upload service |

---

## 2. Prisma Schema

```prisma
// ============ SETTINGS MODULE ============

enum ViewKind {
  ISSUES
  PROJECTS
}

enum ViewMode {
  LIST
  KANBAN
}

model ViewPreference {
  id          String   @id @default(cuid())
  userId      String
  workspaceId String
  kind        ViewKind
  view        ViewMode @default(LIST)   // decided: LIST default (kanban opt-in)
  updatedAt   DateTime @updatedAt

  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@unique([userId, workspaceId, kind]) // one preference per (user, workspace, kind)
}
```

**Profile/theme (existing auth table — fields already in the auth data model):**
`User.name` (1–64) · `User.image` (R2 signed URL | null) · `User.theme` (`light | dark | system`, default `system`).

---

## 3. Field Notes & Design Rationale

- **`ViewPreference` upsert** — `@@unique([userId, workspaceId, kind])` + `ON CONFLICT DO UPDATE` on write: toggling List⇄Kanban never grows the table (one row per kind).
- **`kind` enum, two independent rows** — issues and projects keep separate choices (PRD); a new entity type (cycles?) later adds a kind value, no migration churn.
- **`default(LIST)`** — kanban is opt-in per the decided default.
- **No avatar binary in the DB** — only the signed R2 URL; the object lives in the private bucket (uploads-through-API pattern, consistent with workspace icons).
- **No notification-preference or locale tables** — both post-MVP (PRD future list).

---

## 4. Indexes & Constraints Summary

| Object | Type | Why |
|---|---|---|
| `ViewPreference(userId, workspaceId, kind)` | UNIQUE | Upsert key — one row per kind |
| `ViewPreference(userId, workspaceId)` | INDEX (prefix of the unique) | "All my prefs in this workspace" read |

The table is tiny (2 rows/user/workspace) — no further tuning needed.

---

## 5. Data Lifecycle

| Event | SQL-level behavior |
|---|---|
| Toggle List⇄Kanban | Upsert `ViewPreference` (userId, workspaceId, kind, view) — one statement |
| Read prefs (render time) | Two-row lookup; missing row → default LIST (no backfill writes) |
| Profile update | UPDATE `User.name` / `User.image` — account-wide by construction |
| Theme change | UPDATE `User.theme` — account-wide |
| Avatar upload | Multipart → validate (PNG/JPEG/WebP ≤ 1MB) → PUT R2 (private) → UPDATE `User.image` = signed URL |
| Avatar removal | UPDATE `User.image = null` (R2 object orphaned — cleanup job post-MVP) |
| Workspace deleted | ViewPreference rows cascade |
| User deleted | ViewPreference rows cascade |

---

## 6. Sizing & Free-Tier Fit

Two rows per user per workspace — even 1k users × 10 workspaces ≈ 20k rows ≈ 5MB. Trivial. Avatar objects live in R2's free 10GB, not the DB.

---

## 7. Decisions Adopted (from domain model open questions)

| # | Question | Decision |
|---|---|---|
| 1 | Avatar replace/delete | Replace = new upload (old object orphaned; cleanup job post-MVP); delete = clear field |
| 2 | Theme "system" detection | Client-side `prefers-color-scheme`; server stores the choice only |
| 3 | View preference default | **LIST** for both kinds (kanban opt-in) |
