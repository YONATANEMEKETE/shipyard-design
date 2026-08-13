# Members — Data Model

**Module:** `apps/api/src/modules/members`
**Status:** Draft v0.1 — 2026-08-12
**Stack:** Prisma + PostgreSQL (Neon prod / local container dev)
**PRD source:** §5.3 Members · §6 Permissions & Roles

---

## 1. Overview

Members owns **two tables** — `Membership` (fully defined here; stubbed in the workspace data model) and `Invitation`. Two DB-level guarantees are implemented with **Postgres partial unique indexes** (see §4) — the kind of invariant that belongs in the database, not just the service layer.

| Table | Domain entity | Owner module |
|---|---|---|
| `Membership` | Who belongs, with which role | members ✅ |
| `Invitation` | Pending offer to join | members ✅ |
| `Workspace.ownerId` | Owner reference | workspace (sync contract here) |
| `Project.ownerId` | Project ownership | projects (members triggers transfers) |

---

## 2. Prisma Schema

```prisma
// ============ MEMBERS MODULE ============

enum Role {
  OWNER
  ADMIN
  MEMBER
}

enum InvitationStatus {
  PENDING // expiry is DERIVED from expiresAt — see §3.3
  ACCEPTED
  DECLINED
  REVOKED
}

model Membership {
  id          String   @id @default(cuid())
  workspaceId String
  userId      String
  role        Role     @default(MEMBER)
  createdAt   DateTime @default(now()) // join date (directory)
  updatedAt   DateTime @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  user      User      @relation(fields: [userId], references: [id])

  @@unique([workspaceId, userId])
  @@index([userId])        // "my workspaces" (switcher, auto-enter)
  @@index([workspaceId])   // member directory
}

model Invitation {
  id          String           @id @default(cuid())
  workspaceId String
  email       String // normalized: trimmed + lowercased
  role        Role             @default(MEMBER) // NEVER OWNER (domain rule 5)
  token       String           @unique // stored HASHED; plaintext only in the invite link
  status      InvitationStatus @default(PENDING)
  invitedById String
  invitedAt   DateTime         @default(now())
  expiresAt   DateTime         // 7 days default (decided)
  acceptedAt  DateTime?
  declinedAt  DateTime?
  revokedAt   DateTime?
  createdAt   DateTime         @default(now())
  updatedAt   DateTime         @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  invitedBy User      @relation(fields: [invitedById], references: [id])

  @@index([email])             // "invitations for this address" (accept lookup)
  @@index([workspaceId])       // pending list in the directory
  @@index([status, expiresAt]) // expiry sweep
}
```

**Plus two partial unique indexes added in the migration SQL** (Prisma cannot express these):

```sql
-- EXACTLY ONE OWNER per workspace, enforced by the database:
CREATE UNIQUE INDEX membership_single_owner
  ON "Membership" (workspace_id) WHERE role = 'OWNER';

-- ONE pending invitation per (workspace, email) — re-invites after
-- accept/decline/revoke/expiry are legal (history preserved):
CREATE UNIQUE INDEX invitation_one_pending
  ON "Invitation" (workspace_id, email) WHERE status = 'PENDING';
```

---

## 3. Field Notes & Design Rationale

### 3.1 Membership
- **`@@unique([workspaceId, userId])`** — one membership per user per workspace; the domain's most sacred constraint.
- **No `joinedAt`-style extra state** — `createdAt` doubles as the join date for the directory.
- **`role` enum, not string** — typos are compile errors; the RBAC guards switch on it.
- **No denormalized "status"** — a member is a row; absence of the row IS the removed state.

### 3.2 Invitation
- **`token` hashed at rest** — same pattern as session tokens (auth data model §3.2): the plaintext token lives only in the invite link (`/invite?token=...`); the DB stores a hash. A DB leak cannot mint invitations.
- **`invitedById`** — authorization anchor for resend/revoke (only the inviter or an Owner may act on a pending invite) and directory context ("invited by").
- **`role` limited by the service** — the enum allows OWNER, but the invite service rejects it (domain rule 5); the DB cannot express "never OWNER here", so the guard lives in one place: the invite service.
- **Terminal states keep the row** — accept/decline/revoke set a timestamp + status instead of deleting. History survives (audit-friendly); the partial unique index keeps only *pending* invites unique, so re-inviting after any terminal state works.

### 3.3 Expiry model
- **Expiry is derived:** `status = PENDING AND expiresAt < now()` = expired. No background job needed for correctness — reads treat it as dead on sight.
- The optional **expiry sweep** (future worker) can materialize `status = EXPIRED` rows for hygiene; correctness never depends on it.

---

## 4. Indexes & Constraints Summary

| Object | Type | Why |
|---|---|---|
| `Membership(workspaceId, userId)` | UNIQUE | One membership per user per workspace |
| `Membership(workspace_id) WHERE role = 'OWNER'` | **partial UNIQUE** | Exactly one Owner per workspace — DB-enforced |
| `Membership.userId` | INDEX | "My workspaces" (switcher hot path) |
| `Membership.workspaceId` | INDEX | Member directory |
| `Invitation.token` | UNIQUE | Token lookup on accept (O(1)) |
| `Invitation(workspace_id, email) WHERE status = 'PENDING'` | **partial UNIQUE** | One pending invite per email per workspace; re-invite legal after terminal states |
| `Invitation.email` | INDEX | "Invitations for this address" |
| `Invitation.workspaceId` | INDEX | Pending list |
| `Invitation(status, expiresAt)` | INDEX | Expiry sweep |

---

## 5. DB-Enforced vs Service-Enforced Invariants

| Invariant | Enforced by |
|---|---|
| One membership per (workspace, user) | **DB** (unique) |
| Exactly one Owner per workspace | **DB** (partial unique index) |
| One pending invitation per (workspace, email) | **DB** (partial unique index) |
| Owner never granted by invitation | **Service** (invite guard) |
| Atomic ownership transfer (ownerId + both roles) | **Service** (single Prisma transaction) |
| Project auto-transfer on remove/leave | **Service** (same transaction via projects service) |
| Admin cannot invite/remove Admins or Owners | **Service** (role guards) |
| Removed member loses access immediately | **Service** (per-request membership check) |

The DB covers what Postgres can express; everything else is enforced in exactly one service location each.

---

## 6. Data Lifecycle

| Event | SQL-level behavior |
|---|---|
| Invite | INSERT `Invitation` (PENDING, hashed token, +7d) + Resend email with link |
| Resend | UPDATE `token` (new hash) + `expiresAt` (refresh window) — same row, still PENDING |
| Revoke | UPDATE `status = REVOKED`, `revokedAt = now` |
| Decline | UPDATE `status = DECLINED`, `declinedAt = now` |
| Expire (derived) | No write — reads treat PENDING + past `expiresAt` as dead; sweep may materialize |
| **Accept** | TRANSACTION: INSERT `Membership` (offered role) + UPDATE `Invitation` (ACCEPTED, `acceptedAt`) — both or neither |
| **Remove member** | TRANSACTION: UPDATE owned `Project.ownerId → workspace Owner` (via projects service) + DELETE `Membership` |
| **Leave workspace** | Same transaction as remove (member-initiated) |
| **Transfer ownership** | TRANSACTION: UPDATE `Workspace.ownerId` + UPDATE recipient `Membership.role = OWNER` + UPDATE transferor `Membership.role = ADMIN` |
| **Change role** | UPDATE `Membership.role` (Owner only) — partial index still holds: only the Owner row may carry OWNER |
| Workspace deleted | CASCADE removes memberships + invitations (workspace contract) |

**Failure semantics (PRD):** if any transaction above fails, nothing changes — membership, roles, and project ownership remain exactly as before.

---

## 7. Cross-Module Write Contracts

| Contract | Implementation |
|---|---|
| `Workspace.ownerId` sync | Transfer transaction writes it (workspace data model §4); members service is the only writer |
| Project auto-transfer | `projectsService.transferOwnedProjects(userId → workspaceOwnerId)` called *inside* the remove/leave transaction — includes archived projects (domain rule 9) |
| Verification gate | Accept flow checks `User.emailVerified` before the transaction starts (auth module state, read-only here) |
| Notifications (post-MVP) | Invite-accepted events can be added later without schema change (status + timestamps already recorded) |

---

## 8. Sizing & Free-Tier Fit

Membership rows ~200 bytes; invitations ~300 bytes. Even 1,000 workspaces × 50 members + churn ≈ single-digit MB — negligible inside Neon's 0.5GB free tier. All hot queries are covered by the indexes above.

---

## 9. Decisions Adopted (from domain model open questions)

| # | Question | Decision |
|---|---|---|
| 1 | Invitation expiry | **7 days**, configurable constant |
| 2 | Re-invite removed/declined users | **Yes** — enabled by the partial unique index (history kept, new pending invite allowed) |
| 3 | Decline UI | Decline exists in the API; web shows it in the invite screen — confirm placement during implementation |
| 4 | Member count limits | **None** (PRD default) |
