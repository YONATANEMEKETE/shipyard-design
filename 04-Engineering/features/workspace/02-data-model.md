# Workspace — Data Model

**Module:** `apps/api/src/modules/workspace`
**Status:** Draft v0.1 — 2026-08-12
**Stack:** Prisma + PostgreSQL · relations to `members` / `projects` / `issues` / `cycles`
**PRD source:** §5.2 Workspace

---

## 1. Overview

Workspace owns **one table** (`Workspace`) plus the cascade contract that makes deletion atomic. The `Membership` table is owned by the `members` module — it is shown here only for the relation and the sync contract with `Workspace.ownerId`.

| Table | Domain entity | Owner module |
|---|---|---|
| `Workspace` | The container | workspace ✅ |
| `Membership` | Who can enter, with which role | members (defined there) |

**Identity rule reflected in schema:** no unique constraint on `name` (duplicates are legal); everything references `Workspace.id`.

---

## 2. Prisma Schema

```prisma
// ============ WORKSPACE MODULE ============

enum WorkspaceStatus {
  ACTIVE
  ARCHIVED // DELETED is not a state — the row is removed
}

model Workspace {
  id         String          @id @default(cuid())
  name       String // display label only — duplicates allowed, never an identifier
  icon       String? // R2 object URL (private bucket, signed access)
  status     WorkspaceStatus @default(ACTIVE)
  archivedAt DateTime? // set on archive, cleared on restore
  ownerId    String // denormalized Owner reference (see §4 sync contract)
  createdAt  DateTime        @default(now())
  updatedAt  DateTime        @updatedAt

  owner      User          @relation("WorkspaceOwned", fields: [ownerId], references: [id])
  members    Membership[]
  projects   Project[]
  issues     Issue[]
  cycles     Cycle[]

  @@index([ownerId]) // "workspaces I own"
  @@index([status, archivedAt]) // archived list queries
}

// ============ MEMBERSHIP (owned by members module — reference only) ============

model Membership {
  id          String   @id @default(cuid())
  workspaceId String
  userId      String
  role        Role     @default(MEMBER) // OWNER | ADMIN | MEMBER (defined in members)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  user      User      @relation(fields: [userId], references: [id])

  @@unique([workspaceId, userId])
  @@index([userId]) // "my workspaces" (switcher, auto-enter)
  @@index([workspaceId]) // member directory
}

// ============ RELATION CONTRACT (owned tables define these) ============

// Project / Issue / Cycle / Invitation:  workspaceId String  + onDelete: Cascade
// (defined fully in their modules' data models)
```

---

## 3. Field Notes & Design Rationale

- **`name` — no unique index, not even case-insensitive.** Duplicate names are a first-class feature (PRD §5.2); the switcher disambiguates with icon + role. If search ever needs name matching, a plain B-tree index is added then.
- **`status` + `archivedAt`** — the Shipyard archival pattern (shared with projects/issues/cycles): state machine lives in the status enum; the timestamp answers "when". `restore` = `status = ACTIVE` + `archivedAt = null`.
- **No `DELETED` status** — permanent deletion removes the row; everything below it dies via cascade (§5).
- **`ownerId` denormalized** — one Owner is queried constantly (dashboard ownership, switcher "owner" badge, project auto-transfer targets). Deriving it from `Membership(role=OWNER)` would force a join on every query for zero gain. The cost is a **sync contract**: every role/ownership change must update both sides in one transaction (§4).
- **`icon` as R2 URL** — object stored in the private bucket, served via signed URL (uploads-through-API pattern, ADR-004). No binary in the DB.
- **No `slug`** — routing is id-based (domain rule 2). URLs look like `/workspaces/{id}/...`.

---

## 4. Sync Contract: `ownerId` ↔ Membership

The Owner exists in two places by design:

| Event | Transaction (single) |
|---|---|
| Workspace created | INSERT `Workspace` (ownerId = creator) **+** INSERT `Membership` (role = OWNER) — both or neither |
| Ownership transferred | UPDATE `Workspace.ownerId` = recipient **+** UPDATE recipient `Membership.role = OWNER` **+** UPDATE old owner `Membership.role = ADMIN` |
| Owner leaves/removed | Blocked until transfer (domain rule); enforced by the members service, not the schema |

`ownerId` and `Membership(role=OWNER)` must never disagree — the invariant is protected by running every mutation through the members service transaction. Prisma alone cannot express it; the service is the guard.

---

## 5. Deletion Cascade Contract

`DELETE FROM Workspace WHERE id = $1` is **atomic by the database** — no app-level orchestration:

| Table | FK | onDelete |
|---|---|---|
| `Membership` | workspaceId | CASCADE |
| `Invitation` (members) | workspaceId | CASCADE |
| `Project` | workspaceId | CASCADE |
| `Cycle` | workspaceId | CASCADE |
| `Issue` | workspaceId | CASCADE |
| `Comment` (via issue) | issueId | CASCADE |
| `Notification` (issue-linked) | issueId / workspaceId | CASCADE (dead references must not survive) |
| `User` / auth tables | — | **untouched** (domain rule 10) |

**Guarantees:**
- One DELETE statement, one Postgres transaction — all-or-nothing by ACID (a failed delete leaves the Archived workspace unchanged, domain rule 11).
- User accounts and their data in other workspaces are untouched.
- The exact-name confirmation gate happens **before** the DELETE is issued (application layer), not inside the transaction.

---

## 6. Data Lifecycle

| Event | SQL-level behavior |
|---|---|
| Create (with owner) | TRANSACTION: INSERT `Workspace` + INSERT `Membership(OWNER)` |
| Rename / icon change | UPDATE `Workspace` (name / icon) |
| Archive | UPDATE `status = ARCHIVED`, `archivedAt = now` |
| Restore | UPDATE `status = ACTIVE`, `archivedAt = null` |
| Permanent delete | DELETE `Workspace` → cascade removes memberships, invitations, projects, cycles, issues, comments, notifications |
| Ownership transfer | See §4 (single transaction, both sides) |
| Auto-enter / switcher | SELECT memberships (by userId) JOIN workspaces — ordered by `updatedAt` |

**Read paths:**
- Switcher: `Membership WHERE userId = $1` → workspace rows (name, icon, role, status) — includes Archived workspaces for the Owner.
- Archived list: `Workspace WHERE ownerId = $1 AND status = ARCHIVED`.
- Scoping anchor for every other module: `workspaceId` from the route + membership check — always the same indexed lookup.

---

## 7. Sizing & Free-Tier Fit

Workspace + membership rows are sub-kilobyte. Even 1,000 workspaces with 50 members each ≈ a few MB. Negligible inside Neon's 0.5GB free tier. The `Membership(userId)` and `Membership(workspaceId)` indexes are the only hot paths and are covered.

---

## 8. Decisions Adopted (from domain model open questions)

| # | Question | Decision |
|---|---|---|
| 1 | Workspace count per user | **Unlimited** (PRD default) — no quota in MVP |
| 2 | Icon upload constraints | **PNG/JPEG/WebP, ≤ 1MB**, validated at the API edge; stored in R2 private bucket via the uploads-through-API pattern; signed URL in `icon` |
| 3 | Archived URL for non-owners | Redirect to the active workspace dashboard with a notice (API design doc) |
| 4 | Onboarding skip | No escape hatch — every login returns to onboarding until a workspace exists (PRD flow 3) |
