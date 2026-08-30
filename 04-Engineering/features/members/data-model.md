# Members — Data Model

**Status:** Approved for implementation (F3)
**Last updated:** 2026-08-30
**Sources:** `features/members/spec.md` · `features/workspace/spec.md` · `features/workspace/data-model.md` (F2 precedent — membership seed, partial owner index, cascade convention) · `features/auth/data-model.md` (identity — `user` table, `emailVerified` gate) · `00-architecture.md` §5, §8, §9 · `ADR-001` (Prisma + Postgres + Better Auth conventions) · `ADR-002` (shared contracts) · `Implementation Plan.md` F3
**Owner:** `apps/api` — Prisma-owned (hand-modeled, like workspace). Every model here follows Better Auth's conventions where sensible so the whole schema reads consistently.

---

## 1. Overview

Members owns **who may cross a workspace boundary and with what role**, plus the token-gated door that grants entry: invitations. It is the only feature that may create, change, or remove workspace access.

Three persistence concerns, two tables touched:

| Concern | Table | Formalized by |
|---|---|---|
| The door (who belongs, with which role) | `workspace_member` | **Seeded in F2**, widened by F3 |
| The invitation to cross it | `invitation` | **F3 (this milestone)** |
| The human behind both | `user` (Better Auth) | F1 — read-only here, joined for directory and verified-gate |

Workspace itself (`workspace`) is owned by F2 and is read here only to scope queries and enforce the frozen-archived guard. Projects are owned by F4 — F3 documents the `transferOwnedProjects` contract but does not create project tables yet (see §7).

All workspace-scoped access decisions in every later feature call the membership rows defined here; the invitation rows defined here are the only way a non-member becomes a member.

---

## 2. Core schema (Prisma-owned)

### 2.1 `workspace_member` — final shape (F2 seed → F3 widening)

One row per user per workspace — the only proof of access (spec rule 5, §5.2). F2 wrote only `role = OWNER` with `MEMBER` reserved for read-only deserialization; F3 widens the enum to include `ADMIN` without any data migration beyond the enum alteration.

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK `@default(cuid())` | |
| `workspaceId` | `String` | FK → `workspace.id` `onDelete: Cascade` + `@@index([workspaceId])` | Deleting a workspace removes all memberships (spec workspace rule 8). |
| `userId` | `String` | FK → `user.id` `onDelete: Cascade` + `@@index([userId])` | Points at Better Auth's `user`. Powers "list my workspaces", role checks, and transfer target validation. |
| `role` | `WorkspaceRole` | — | `OWNER \| ADMIN \| MEMBER` after F3. `OWNER` never granted by invitation — F3 service constraint, documented here for the trail. |
| `createdAt` | `DateTime` | `@default(now())` | Join date — history preserved through archive/restore of the parent workspace. |

```prisma
enum WorkspaceRole {
  OWNER
  ADMIN   // F3 adds — additive widening, existing rows untouched
  MEMBER
}

model WorkspaceMember {
  id          String        @id @default(cuid())
  workspaceId String
  userId      String
  role        WorkspaceRole @default(MEMBER)
  createdAt   DateTime      @default(now())

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([workspaceId, userId]) // one membership row per user per workspace — independent per workspace
  @@index([workspaceId])
  @@index([userId])
  @@map("workspace_member")
}
```

The exactly-one-Owner invariant established in F2 continues to hold via the partial unique index (no denormalized `workspace.ownerId` column — see workspace `data-model.md` §3 D1, unchanged):

```sql
CREATE UNIQUE INDEX workspace_single_owner
  ON workspace_member (workspace_id) WHERE role = 'OWNER';
```

Creation (F2) and F3's ownership transfer swap owner rows within a transaction so the constraint is the source of truth, not application discipline.

### 2.2 `invitation` — the only door for non-members (F3)

One row per pending (or recently resolved) invitation. A pending row grants **no access** — accepting inside a transaction is the only way it becomes a `workspace_member` row (spec §3.2).

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK `@default(cuid())` | Internal row id; never exposed as the acceptance secret. |
| `workspaceId` | `String` | FK → `workspace.id` `onDelete: Cascade` + `@@index([workspaceId])` | Deleting a workspace removes pending invitations (workspace data-model §7 convention). |
| `email` | `String` | `@db.VarChar(320)` | Normalized invitee address: **trimmed + lowercased** server-side before persist. Comparison is case-insensitive via normalization, not via DB collation. |
| `role` | `WorkspaceRole` | — | The role the invitee will receive on accept: `MEMBER` or `ADMIN` only. `OWNER` is rejected at the service layer (spec §3.2) even though the enum technically allows it. |
| `token` | `String` | `@unique` | Opaque acceptance secret embedded in the email link (`/invite/:token`). Generated with `crypto.randomBytes(32)` → hex/base62, collision retry on `P2002`. Single-use: deleted or marked on accept. `@db.Text` length not bounded by varchar — stored as `@db.VarChar(128)` in practice, treated as opaque. |
| `status` | `InvitationStatus` | `@default(PENDING)` | Lifecycle state. See enum and §6.2. |
| `expiresAt` | `DateTime` | `@@index([expiresAt])` | `createdAt + INVITATION_TTL` (default 7 days, configurable via env — spec §7 Q1). Checked on accept; expired rows are rejected, not silently accepted. |
| `createdById` | `String?` | FK → `user.id` `onDelete: SetNull` + `@@index([createdById])` | Inviting user. `SetNull` — the invitation survives if the inviter leaves, so it can still be accepted or revoked. Nullable for that reason. |
| `createdAt` | `DateTime` | `@default(now())` | |
| `updatedAt` | `DateTime` | `@updatedAt` | Revoke/accept/decline/resend touch this; resend does **not** rotate `token` — it re-sends the same link and bumps `updatedAt`. |

```prisma
enum InvitationStatus {
  PENDING   // awaiting action — the only status that counts as "pending" for uniqueness
  ACCEPTED  // consumed — membership row created in same transaction
  REVOKED   // cancelled by Owner/Admin before accept
  DECLINED  // explicitly declined by invitee
  EXPIRED   // lazily marked or left as PENDING + expired — see §6.2; service treats both as expired
}

model Invitation {
  id          String           @id @default(cuid())
  workspaceId String
  email       String           @db.VarChar(320)
  role        WorkspaceRole
  token       String           @unique
  status      InvitationStatus @default(PENDING)
  expiresAt   DateTime
  createdById String?
  createdAt   DateTime         @default(now())
  updatedAt   DateTime         @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  createdBy User?     @relation(fields: [createdById], references: [id], onDelete: SetNull)

  @@index([workspaceId])
  @@index([email])
  @@index([expiresAt])
  @@index([createdById])
  @@map("invitation")
}
```

`Workspace` and `User` gain the back-relation fields:

```prisma
model Workspace {
  // ... existing F2 fields
  members     WorkspaceMember[]
  invitations Invitation[]
}

model User {
  // ... existing F1 fields + workspaceMembers
  sentInvitations Invitation[] @relation("CreatedInvitations")
}
```

> The relation name on `User.sentInvitations` is explicit to avoid ambiguity with `workspaceMembers`. The Prisma field on `Invitation.createdBy` carries `@relation("CreatedInvitations")` to match.

### 2.3 Token handling

- `token` is the bearer secret in the email URL (e.g. `/invite/:token` → `GET /api/v1/invitations/:token` for preview + `POST .../accept`). It is **not** a JWT — just an opaque random string.
- Stored as a raw unique string for MVP. Hashing the token at rest (store `tokenHash`, compare `hash(raw)`) is a hardening step deferred to post-MVP; the uniqueness guarantee and single-use consumption already prevent replay without hashing.
- Clock source for `expiresAt` is the API server's `now()` at creation; acceptance checks `expiresAt > now()` inside the accepting transaction to avoid TOCTOU.

---

## 3. Key decisions & alternatives

### D1 — `WorkspaceRole` widened additive, no data rewrite

**Decision:** Extend the enum from `OWNER | MEMBER` (F2) to `OWNER | ADMIN | MEMBER` in place. Existing rows keep their values; no `UPDATE workspace_member` migration. The F2 `MEMBER` variant already existed so non-owner deserialization never broke — `ADMIN` simply becomes a third legal value.

*Considered and rejected:* a separate `role` table or string column without an enum — loses the DB-level guard that only the three legal roles can be persisted, and makes the partial owner index harder to reason about.

### D2 — Pending uniqueness is a **partial unique index**, not a table-wide unique

**Decision:** At most one `PENDING` invitation per `(workspaceId, normalized email)`:

```sql
CREATE UNIQUE INDEX invitation_single_pending
  ON invitation (workspace_id, lower(email)) WHERE status = 'PENDING';
-- Prisma migration: raw SQL in the migration folder (Prisma @@unique cannot express partial conditions).
-- Application additionally lowercases email before persist, so the lower() is defense-in-depth;
-- the effective uniqueness is on the already-lowercased value.
```

This satisfies the behavioral rules (spec §3.2):
- Cannot invite an email that already has a pending invitation (resend instead).
- Re-inviting after `ACCEPTED | REVOKED | DECLINED | EXPIRED` **is** allowed — those rows no longer satisfy `WHERE status='PENDING'`.
- Cannot invite yourself or an existing member — those are service-layer checks against `user.email` and `workspace_member`, not DB constraints, so the error shape can distinguish the cases.

*Considered and rejected:* `@@unique([workspaceId, email])` globally — would block re-inviting a previously removed user (§7 Q2) and would require deleting history rows to re-invite, losing auditability.

### D3 — No membership `status` column

**Decision:** Membership is existence of a `workspace_member` row. There is no `INVITED | ACTIVE | SUSPENDED` status on the membership table. A pending invitation is **not** a membership row; it lives only in `invitation`. This keeps `requireWorkspaceMember` a single existence check and prevents the "pending member shows up in directory" bug.

### D4 — Invitation `role` reuses `WorkspaceRole` but service restricts it

**Decision:** The column type is `WorkspaceRole` so the DB enum stays single-sourced, but the service layer rejects `OWNER` on create. An `ADMIN` invitation is only creatable by an `OWNER`; a `MEMBER` invitation by `OWNER` or `ADMIN` (permission matrix, spec §3.1). Doing this at the service layer lets error codes distinguish "forbidden role assignment" (`FORBIDDEN_ROLE`) from a DB enum violation, and keeps the door open for a future check constraint `CHECK (role != 'OWNER')` without a Prisma enum split.

### D5 — Token is `@unique`, not derived from `id`

**Decision:** `id` is the internal primary key; `token` is the external secret. They are distinct so the email link cannot be guessed from a sequential or cuid `id`, and so rotating or re-sending strategy can be changed (F3 resends the same token; a future "rotate on resend" would generate a new token without changing `id`).

### D6 — `createdById` is `SetNull`, not `Cascade`

**Decision:** If the inviting user leaves, the invitation remains usable and revocable by any remaining `OWNER`/`ADMIN` with permission. `Cascade` would silently delete pending invitations when their creator departs, which would hide them from the pending list and break the resend/revoke UX.

### D7 — Expiry is lazy + transactional, not a cron flip

**Decision:** A `PENDING` row whose `expiresAt` is in the past is treated as expired at accept time (`expiresAt > now()` check inside the transaction). A background job that flips `PENDING → EXPIRED` is optional post-MVP for list hygiene; the correctness guarantee does not depend on it. This avoids a required worker in the MVP (architecture §11 — no queues/workers).

### D8 — Project ownership transfer is a service contract, not a table here

**Decision:** This data model does **not** create `project` tables. The behavior "remove/leave atomically transfers owned projects to the Workspace Owner" is fulfilled in F3 Checkpoint B by calling `projectsService.transferOwnedProjects(workspaceId, fromUserId, toOwnerUserId, tx)` inside the same Prisma transaction that deletes the membership row. The contract is documented here so the F4 migration can implement it without revisiting F3 decisions. See §7.

---

## 4. Shared contracts (`packages/shared`)

Added (or widened) in F3, consumed by `api` and `web` (ADR-002). All schemas mirror the Prisma enums above.

```ts
// zod enums — mirror Prisma enums §2
export const workspaceRoleSchema = z.enum(["OWNER", "ADMIN", "MEMBER"]); // widened from ["OWNER","MEMBER"]
export const invitationStatusSchema = z.enum(["PENDING", "ACCEPTED", "REVOKED", "DECLINED", "EXPIRED"]);

// canonical invitation TTL — keep in sync with api env INVITATION_TTL_DAYS (default 7)
export const INVITATION_TTL_DAYS = 7;

// request contracts owned by the members module
export const inviteMembersSchema = z.object({
  emails: z.array(z.string().trim().toLowerCase().email()).min(1).max(20),
  role: z.enum(["MEMBER", "ADMIN"]), // OWNER never offered — service rejects ADMIN when caller is ADMIN
});

export const resendInvitationSchema = z.object({ invitationId: z.string().cuid() });
export const revokeInvitationSchema = z.object({ invitationId: z.string().cuid() });

export const changeMemberRoleSchema = z.object({
  memberId: z.string().cuid(),
  role: z.enum(["MEMBER", "ADMIN"]), // OWNER excluded — only via transfer-ownership
});

export const removeMemberSchema = z.object({ memberId: z.string().cuid() });

export const transferOwnershipSchema = z.object({
  targetMemberId: z.string().cuid(), // must be a current workspace_member row in same workspace
});

// response contracts
export const workspaceMemberCardSchema = z.object({
  id: z.string(),            // workspace_member.id (membership row id, not user id)
  userId: z.string(),
  workspaceId: z.string(),
  name: z.string(),
  email: z.string().email(),
  image: z.string().nullable(),
  role: workspaceRoleSchema,
  createdAt: z.string().datetime(), // join date
});

export const invitationCardSchema = z.object({
  id: z.string(),
  workspaceId: z.string(),
  email: z.string().email(),
  role: workspaceRoleSchema, // MEMBER | ADMIN as stored
  status: invitationStatusSchema,
  token: z.string(),         // only returned to privileged readers (Owner/Admin list); never to public preview
  expiresAt: z.string().datetime(),
  createdById: z.string().nullable(),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});

// invitation preview — public, unauthenticated, token-gated (no email leak beyond the token holder)
export const invitationPreviewSchema = z.object({
  workspaceName: z.string(),
  workspaceIcon: z.string().nullable(),
  role: workspaceRoleSchema,
  email: z.string().email(), // the invited address — shown so the user can confirm it matches their account
  expiresAt: z.string().datetime(),
  status: invitationStatusSchema, // so the UI can render expired/revoked without an extra call
});
```

Validation notes:
- `emails` are lowercased and trimmed before persist, matching the partial index normalization (§3 D2).
- `role` on invite is validated twice: Zod restricts to `MEMBER|ADMIN`, then the service checks the caller's `role` can grant it (`ADMIN` cannot grant `ADMIN`).
- `changeMemberRole` and `removeMember` identify the target by `memberId` (membership row id), not `userId`, so the workspace scope is unambiguous without an extra lookup.
- `transferOwnership` takes a `memberId` for the same reason; the service verifies it belongs to the same `workspaceId` and is not already `OWNER`.

---

## 5. Integrity invariants → spec rule mapping

Every behavioral rule in `members/spec.md` §5 (and PRD §5.3/§6 where members is the owner) lands somewhere concrete:

| Spec rule | Enforcement point |
|---|---|
| Every workspace has exactly one Owner | Partial unique index `workspace_single_owner` (§2.1) + transactional transfer (swap) |
| One membership per user per workspace, roles independent per workspace | `@@unique([workspaceId, userId])` on `workspace_member` |
| Role changes and removals take effect immediately | No caching layer; every request re-reads `workspace_member.role` via `resolveWorkspaceContext` / member lookup inside the request transaction |
| Pending invitation grants nothing | No `workspace_member` row exists until `ACCEPTED` transaction inserts it; `requireWorkspaceMember` is the only door |
| Owner role never granted by invitation | `inviteMembersSchema` excludes `OWNER`; service rejects `role=OWNER` even if smuggled |
| Owners invite as Member/Admin, Admins as Member only | Service check: `if caller.role === 'ADMIN' && requestedRole === 'ADMIN' → 403 FORBIDDEN_ROLE` |
| Admins remove Members only; Owner changes roles and transfers ownership | `requireWorkspaceRole` + per-action service checks (see `api-design.md`) |
| Owner cannot leave or be removed before transfer | Service preconditions: `if target.role === 'OWNER' → 409 CANNOT_REMOVE_OWNER`; `if caller.role === 'OWNER' on leave → 409 TRANSFER_REQUIRED` |
| Owned projects transfer to Workspace Owner on remove/leave (same operation, fixed recipient, archived included) | F3 Checkpoint B transaction: `delete membership` + `projectsService.transferOwnedProjects(...)` in one Prisma `$transaction`; recipient resolved as the current `OWNER` member of the same workspace, never caller-supplied |
| Invitations expire, can be revoked, cannot be accepted twice | `expiresAt` check + `status` guard inside accepting transaction; `REVOKED`/`ACCEPTED`/`DECLINED` rows never transition back to `PENDING` |
| Invitation acceptance requires verified account | Service checks `user.emailVerified === true` (Better Auth `user` row) before inserting membership; unverified caller receives `403 EMAIL_NOT_VERIFIED` with resend path |
| Transfer/remove/leave are all-or-nothing | Single Prisma `$transaction` per operation; any failure rolls back membership, role, and project-transfer writes together |

Integrity summary — constraints added or relied upon in F3:

| Constraint | Where | Purpose |
|---|---|---|
| `workspace_single_owner` partial unique | `workspace_member` | Exactly one OWNER per workspace |
| `@@unique([workspaceId, userId])` | `workspace_member` | One membership per user per workspace |
| `@@unique(token)` | `invitation` | Single-use, unguessable acceptance link |
| `invitation_single_pending` partial unique | `invitation` | At most one pending invite per (workspace, email) — resend instead |
| `@@index([workspaceId])` | both tables | Workspace-scoped list hot path |
| `@@index([userId])` / `@@index([createdById])` | `workspace_member` / `invitation` | User-centric lookups + invite audit |
| `@@index([expiresAt])` | `invitation` | Expiry filtering and cleanup |
| `onDelete: Cascade` (workspace → children) | FKs | Deleting a workspace removes memberships and invitations (spec workspace rule 8) |
| `onDelete: Cascade` (user → membership) | `workspace_member.userId` | Deleting a user (future admin) removes their memberships atomically |
| `onDelete: SetNull` (user → invitation) | `invitation.createdById` | Invitations outlive their creator |

---

## 6. Lifecycle semantics at the data layer

### 6.1 Membership lifecycle

- **Create via invitation accept:** single transaction reads `invitation` by `token` with `FOR UPDATE` (Prisma serializable via `$transaction` + unique lookup), validates `status === PENDING && expiresAt > now()`, validates the accepting `user` is verified and that no `workspace_member` row already exists for `(workspaceId, userId)` nor a membership for that email in that workspace, then inserts `workspace_member { workspaceId, userId, role: invitation.role }` and updates `invitation.status = ACCEPTED` + `updatedAt = now()`. Any validation failure aborts — no partial membership.
- **Create via workspace creation (F2):** existing path — `workspace + workspace_member { role: OWNER }` in one transaction. Unchanged.
- **Role change (Owner only):** `PATCH member role` — transaction verifies caller is `OWNER`, target is `MEMBER` or `ADMIN` in same workspace, requested role is `MEMBER ↔ ADMIN` (never `OWNER`), then `UPDATE workspace_member SET role`. The partial owner index is never touched by this path, so it cannot violate the single-owner invariant.
- **Ownership transfer (Owner only):** transaction verifies caller is current `OWNER`, target is a different `MEMBER`/`ADMIN` in same workspace and still present, then atomically `UPDATE caller role → ADMIN`, `UPDATE target role → OWNER`. The partial unique index allows the swap only when done in one transaction that never observes two `OWNER` rows simultaneously — Prisma `$transaction` with two updates in sequence satisfies this because the index is checked at commit (deferrable) or per-statement but the intermediate state with two owners would violate it; the correct pattern is `UPDATE target → OWNER` then `UPDATE caller → ADMIN` inside the same transaction with the index as `DEFERRABLE INITIALLY DEFERRED` if the DB enforces per-statement, or a single raw `UPDATE ... CASE` — documented in migration notes. Failure anywhere → full rollback, both retain original roles (spec rule 10).
- **Remove member:** transaction verifies caller permission (`OWNER` can remove `MEMBER`/`ADMIN`; `ADMIN` can remove `MEMBER` only), target is not `OWNER`, then (Checkpoint B) calls `projectsService.transferOwnedProjects(workspaceId, fromUserId=target.userId, toOwnerUserId=currentOwner.userId, tx)` inside the same `$transaction`, then `DELETE FROM workspace_member WHERE id = target.id`. Access is gone on next request (no cache).
- **Leave workspace:** same as remove but caller is the target. Additional precondition: `if caller.role === 'OWNER' → reject with TRANSFER_REQUIRED` — they must transfer first and become `ADMIN`.
- **Cross-workspace safety:** `resolveWorkspaceContext(:slug)` (F2) is the only resolver of workspace context for member routes; a non-member receives the same generic `404 WORKSPACE_NOT_FOUND` whether or not the slug exists — no existence leak.

### 6.2 Invitation lifecycle

```text
PENDING ──accept (verified, not expired, no existing membership)──▶ ACCEPTED
   │──revoke (Owner/Admin with permission)────────────────────────▶ REVOKED
   │──decline (invitee, verified)────────────────────────────────▶ DECLINED
   │──expiresAt passes without action────────────────────────────▶ treated as EXPIRED
   │         (row may stay PENDING; service rejects on use; optional job flips to EXPIRED)
   └──resend (Owner/Admin)──▶ stays PENDING, re-sends same token, bumps updatedAt
```

- **Create:** `POST invite` — validates caller permission, validates each email is not the caller's own, not an existing member's email in that workspace, not already pending (partial index would also reject, but service returns the friendlier `PENDING_EXISTS` code before hitting the DB). Generates `token`, `expiresAt = now + TTL`, inserts one row per email inside a transaction. Email sending via Resend (with local dev capture) happens after commit; a send failure does not roll back the row — the pending list still shows it and resend can retry.
- **Resend:** `POST resend` — verifies row is still `PENDING` and not expired, caller has permission, then re-sends the same `token` link and updates `updatedAt`. No new token, no new row — idempotent. Rate-limited.
- **Revoke:** `POST revoke` — `UPDATE status = REVOKED` where `status = PENDING`. Revoked rows never become `ACCEPTED`; a new invitation is required to re-invite (creates a fresh row, fresh token).
- **Accept:** `POST accept` with `token` — the critical transaction described in §6.1. Requires a verified account (`user.emailVerified`). If the invitee's account email differs from `invitation.email` (case-insensitive), the service still allows acceptance when the authenticated user's email matches the lowercased invite — the invitation is to an address, not a user id, so a user who controls that address (verified) may accept. If the user has no account yet, they register + verify first (Auth flow), then the invitation is resumed automatically on the client — the data layer just sees a verified user presenting a valid token.
- **Decline:** `POST decline` with `token` — `UPDATE status = DECLINED` where `status = PENDING` and token holder is the invitee (verified). Explicit decline is separate from expiry; both end the pending state.
- **Expired handling:** `PENDING` + `expiresAt <= now()` is rejected on accept with `INVITATION_EXPIRED`. An optional periodic job or list-time filter may mark such rows `EXPIRED` for display, but correctness never depends on that job.
- **Race safety:** acceptance uses the `@unique(token)` lookup plus a `status = PENDING` guard in the `UPDATE`; two concurrent accepts on the same token — one wins, the other sees `status != PENDING` and receives `INVITATION_ALREADY_USED`.

### 6.3 Archived workspace interaction

`workspace.status = ARCHIVED` flips the whole workspace read-only except lifecycle exits (workspace `api-design.md` §6). For members:

- `GET` (directory, invitation list, preview) — allowed, read-only.
- Any write (`invite`, `resend`, `revoke`, `changeRole`, `remove`, `leave`, `transferOwnership`, `accept`, `decline`) — rejected with `409 WORKSPACE_ARCHIVED` via the shared `resolveWorkspaceContext({ rejectArchived: true })` guard. The data layer never needs to check `status` itself, but service defense-in-depth reasserts it.

---

## 7. Cascade deletion & cross-module contracts

How `workspace` children grow — F3 extends the convention from workspace `data-model.md` §7:

| Table | Introduced | `onDelete` on workspace delete | Notes |
|---|---|---|---|
| `workspace_member` | F2 | `Cascade` | All memberships die with the workspace. |
| `invitation` | **F3** | `Cascade` | Pending (and resolved) invitations die with the workspace. |
| `project` (and its issue unassignment logic) | F4 | `Cascade` | Irrelevant when the whole workspace goes; added here for the transfer contract. |
| `issue`, labels, join tables | F5 | `Cascade` | Direct/descendant children. |
| `notification`, `cycle`, `comment` | F6–F8 | `Cascade` | Every workspace-scoped table without exception. |
| `user` (Better Auth) | F1 | **Never touched** | No deletable path points at accounts; accounts outlive workspaces permanently. |

**Project transfer contract (F3 Checkpoint B ↔ F4):**

```ts
// Owned by projects, called by members. Runs inside the caller's Prisma transaction.
projectsService.transferOwnedProjects(
  workspaceId: string,
  fromUserId: string,
  toOwnerUserId: string,
  tx: Prisma.TransactionClient,
): Promise<number> // count transferred, for the confirmation dialog
```

- Transfers **all** projects where `project.ownerId === fromUserId && project.workspaceId === workspaceId`, including `ARCHIVED` projects — to the current `OWNER` of the same workspace (resolved inside the transaction, never caller-supplied). The count is returned for the Remove/Leave confirmation dialog.
- Called atomically inside the same `$transaction` that deletes the membership row. If either the membership delete or the project transfer fails, neither persists (spec rule 10 — all-or-nothing).
- F3 ships the call site and the guard/permission logic; F4 ships the implementation. Until F4 lands, Checkpoint A tests that remove/leave correctly reject or are hidden where projects would be affected, or that the transfer is a no-op (zero projects) — integration tests covering the cross-module transaction land with Checkpoint B.

Convention for later milestones: adding any workspace-scoped table **requires** `workspaceId` with `onDelete: Cascade` plus an index — treat a missing cascade as a review defect.

---

## 8. Migration workflow

Workspace models are hand-modeled Prisma (unlike F1's Better Auth generation):

```bash
# 1 — widen the enum in schema.prisma (OWNER → OWNER,ADMIN,MEMBER)
# 2 — add the invitation model + back-relations
# 3 — run
pnpm --filter @shipyard/api db:migrate -- --name add_members_and_invitations
pnpm --filter @shipyard/api db:generate
```

- Never hand-edit generated migration SQL afterward (plan §5 Step 3), **except** to add the partial indexes which Prisma's schema language cannot express — those are appended as raw `CREATE UNIQUE INDEX ... WHERE ...` statements inside the same migration folder. This is the same pattern F2 used for `workspace_single_owner`.
- The migration produces: 1 new table (`invitation`), 1 new enum (`InvitationStatus`), 1 widened enum (`WorkspaceRole`), the composite/unique/partial indexes above, and back-relation wiring.
- Widening `WorkspaceRole` is additive — zero data rewrite, existing `OWNER`/`MEMBER` rows untouched.
- The F1 Testcontainers harness applies migrations automatically each test run — F3 tests inherit real-schema validation.

**Post-migration verification (manual, once):**

```sql
-- exactly one owner per workspace still holds
SELECT workspace_id, count(*) FILTER (WHERE role='OWNER') FROM workspace_member GROUP BY workspace_id;
-- pending uniqueness still holds
SELECT workspace_id, lower(email), count(*) FROM invitation WHERE status='PENDING' GROUP BY 1,2 HAVING count(*)>1;
```

---

## 9. What we intentionally do NOT model

| Deferred | Why |
|---|---|
| Custom roles / permission editor | Spec §6 out of scope — only `OWNER/ADMIN/MEMBER` with the fixed matrix. |
| Team groups / org-wide management | Spec §6 — per-workspace only. |
| `project` owner column / issue assignment | Owned by Projects (F4) / Issues (F5); only the transfer contract (§7) is fixed here. |
| Invitation bulk import | Spec §6 — single/batched (≤20) invite only; bulk CSV deferred. |
| SSO / SCIM | Spec §6 — not in MVP. |
| `rateLimit` table for resend | Memory for MVP (like Auth); a table only if limits need persistence. |
| Soft-delete on membership | Deletion is the leave/remove — no trash state; re-invite creates a fresh invitation. |
| Token hashing at rest | Deferred hardening — raw unique token with single-use consumption is sufficient for MVP; hash in place post-MVP without schema change (add `tokenHash` column, dual-read). |
| WebSocket / realtime member presence | Architecture §11 — no workers/queues in MVP; polling or navigation refresh suffices. |

---

## 10. Open product questions — resolved at data layer

| Spec §7 | Decision |
|---|---|
| 1 — invitation expiry period | **7 days default**, configurable via `INVITATION_TTL_DAYS` env (default `7`). `expiresAt` computed as `now + TTL` at creation. No per-invitation TTL override in MVP. |
| 2 — re-invite a previously removed user | **Yes — new invitation.** After `REVOKED`/`DECLINED`/`EXPIRED`/`ACCEPTED` + subsequent removal, the partial pending index no longer blocks that `(workspaceId, email)` pair, so a fresh `PENDING` row with a new token is allowed. The data layer permits it; product confirms it as intended. |
| 3 — decline vs ignore | **Explicit `DECLINED` plus expiry covers it.** No separate "ignore" state. Ignored invitations simply expire; the UI shows `PENDING` until `expiresAt`, then `EXPIRED`. An explicit decline gives the inviter immediate feedback. |
| 4 — member count limits | **None in MVP.** No DB cap; `memberCount` on workspace cards is a derived count (`_count.members`). Revisit quotas post-MVP if needed. |

---

## 11. References

- Shipyard: `features/members/spec.md`, `features/workspace/spec.md`, `features/workspace/data-model.md`, `features/workspace/api-design.md` (guard chain, archived matrix), `features/auth/data-model.md` (user + emailVerified), `00-architecture.md` §5–§9, `ADR-001`, `ADR-002`, `Implementation Plan.md` F3
- Prisma indexes & referential actions: `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- PostgreSQL partial unique index (single-owner + pending-uniqueness): `https://www.postgresql.org/docs/current/indexes-partial.html`
- Better Auth user model & emailVerified gate: `https://www.better-auth.com/docs/concepts/database` + `https://www.better-auth.com/docs/reference/options#emailVerification`

---

*Next artifact: `api-design.md` — endpoint inventory over these two tables, the canonical guard chain per route, error codes/envelopes, and app-flow sequences (Next proxy → Express → guard chain → service → repository).*
