# Members — Domain Model

**Module:** `apps/api/src/modules/members`
**Status:** Draft v0.1 — 2026-08-12
**PRD source:** §5.3 Members · §6 Permissions & Roles · §7.2 Member Rules

---

## 1. Overview & Scope

Members owns **who can cross a workspace's boundary, with what role** — and the rules that keep exactly one Owner in charge: invitations, roles, membership lifecycle, and ownership transfer.

**In scope:**
- Membership (user ↔ workspace, with a role)
- Roles: Owner / Admin / Member + the permission matrix (PRD §6)
- Invitations: create, accept, decline, revoke, resend, expiry
- Membership lifecycle: join, remove, leave, role change, ownership transfer
- The **project auto-transfer** contract when a member leaves or is removed

**Out of scope:**
- Workspace container/lifecycle (archive/restore/delete) → `workspace`
- Identity, verification, sessions → `auth`
- Project ownership mechanics (the *storage* of ownerId on projects) → `projects` — members only *triggers* transfers through the projects service
- Member directory/settings UI → `settings`

**Key separation:** members = **authorization** (what a member can do). auth = **authentication** (who is verified). workspace = the **boundary** (what data lives where).

---

## 2. Domain Entities

### 2.1 Membership

The link between a User and a Workspace carrying exactly one role.

| Attribute | Notes |
|---|---|
| `workspaceId` + `userId` | Unique pair — one membership per user per workspace |
| `role` | `OWNER` · `ADMIN` · `MEMBER` (one role at a time) |
| `joinedAt` | Directory display |

**Invariants:**
- A user may hold memberships in many workspaces; roles are **independent per workspace**.
- Every workspace has **exactly one** membership with role `OWNER`.
- Role changes take effect **immediately** (per-request checks — no cached permissions).

### 2.2 Invitation

A pending, capability-based offer to join a workspace.

| Attribute | Notes |
|---|---|
| `workspaceId`, `email` | Target address |
| `role` | Offered role — fixed at invite time: Owner invites as Member or Admin; Admin invites as **Member only** |
| `token` | Capability token (single-use, expiring) — like auth verification tokens |
| `expiresAt` | Defined period; expired = dead |
| `status` | `PENDING` → `ACCEPTED` · `DECLINED` · `REVOKED` · `EXPIRED` |

**Invariants:**
- Pending invitations grant **no access** (acceptance is the only door).
- Cannot invite: yourself, an existing member, an email with a pending invitation (resend instead), an email already in the workspace.
- Owner role can never be assigned through an invitation.
- A user cannot accept the same invitation twice.

### 2.3 Role (reference model — PRD §6 matrix)

| Capability | Owner | Admin | Member |
|---|:---:|:---:|:---:|
| Manage workspace settings / archive / delete | ✅ | ❌ | ❌ |
| Invite as **Member** | ✅ | ✅ | ❌ |
| Invite as **Admin** | ✅ | ❌ | ❌ |
| Change Member/Admin roles | ✅ | ❌ | ❌ |
| Remove **Members** | ✅ | ✅ | ❌ |
| Remove **Admins** | ✅ | ❌ | ❌ |
| Transfer workspace ownership | ✅ | ❌ | ❌ |
| Create/edit projects, manage cycles | ✅ | ✅ | ❌ |
| Create/edit/archive issues · comment | ✅ | ✅ | ✅ |
| Delete issues | ✅ | ✅ | ❌ |
| View everything | ✅ | ✅ | ✅ |

*Full matrix lives in PRD §6 — this table is the subset members enforces directly.*

---

## 3. Invitation Lifecycle

```
                 ┌──────────┐
   invite ──────▶│ PENDING  │──accept──▶ member (role applied)
                 └────┬─────┘──decline──▶ DECLINED
                      │        ──revoke──▶ REVOKED (by inviter)
                      │        ──timeout─▶ EXPIRED
                      ▼
               (no access at any point before ACCEPTED)
```

**Accept paths:**
- Invitee is an existing verified user → accept directly.
- Invitee is new → register + verify email **first** (auth gate), then accept — the invitation is resumed automatically after verification (PRD flow 7).

---

## 4. Membership Lifecycle

```
                 ┌───────────┐
  accept ───────▶│  MEMBER   │──(Owner changes role)──▶ ADMIN
                 └─────┬─────┘                            │
                       │                                  │
        ┌──────────────┼────────────────┐                 │
        ▼              ▼                ▼                 ▼
   remove (by       leave (own        role change     transfer ownership
   Owner/Admin)     choice)           (Owner only)    (Owner → Member/Admin)
        │              │                │                 │
        └──────────────┴───▶ REMOVED — access gone immediately;
                             owned projects auto-transfer to
                             the Workspace Owner (same operation)
```

**Ownership transfer (atomic):** recipient becomes `OWNER`, transferring Owner becomes `ADMIN` — one transaction, both sides (sync contract with `Workspace.ownerId`, workspace data model §4).

**The Owner's cage:** the Owner cannot leave, be removed, or have their role changed — only transfer, which demotes them to Admin automatically. Then they may leave like any Admin.

---

## 5. Domain Invariants

From PRD §7.2 (Member Rules) + §5.3, condensed:

1. Every membership has exactly one role; roles are per-workspace and independent across workspaces.
2. Every workspace has exactly one Owner.
3. Role changes and removals take effect immediately (per-request enforcement).
4. Removed members immediately lose access; pending invitations grant no access.
5. Owners invite as Member or Admin; Admins invite as Member only; the Owner role is never granted by invitation.
6. Users cannot invite themselves; existing members and duplicate pending invites are blocked.
7. Admins can remove Members but never Admins or Owners; only the Owner changes roles or transfers ownership.
8. The Owner cannot leave or be removed until ownership is transferred; transfer is atomic (recipient → Owner, transferor → Admin).
9. Projects owned by a removed or departing member transfer automatically to the Workspace Owner **in the same operation** — including Archived projects (system-level exception so no project ever retains a former member as owner).
10. An Admin performing a permitted Member removal cannot choose the project-transfer recipient — it is always the Workspace Owner.
11. Invitations expire, can be revoked before acceptance, and cannot be accepted twice.
12. Invitation acceptance requires a verified account (auth gate).

---

## 6. Domain Operations

| Operation | Description | Requires |
|---|---|---|
| `inviteMember` | Create pending invitation (role-limited) | Owner / Admin |
| `resendInvitation` | New token for a pending invite | inviter |
| `revokeInvitation` | Kill a pending invite | inviter |
| `acceptInvitation` | Become a member with the offered role | verified user + valid token |
| `declineInvitation` | Reject the offer | invitee |
| `listMembers` | Member directory (name, email, role, join date, status) | any member |
| `getMember` | Single member + role-appropriate actions (details drawer) | any member |
| `changeRole` | Member ↔ Admin | **Owner** |
| `removeMember` | Remove + auto-transfer owned projects | Owner (Member/Admin targets) / Admin (Member targets only) |
| `leaveWorkspace` | Self-remove + auto-transfer owned projects | Member / Admin (Owner must transfer first) |
| `transferOwnership` | Atomic Owner swap (recipient → Owner, transferor → Admin) | **Owner** |

---

## 7. Cross-Module Contracts

```
auth ──verified identity──▶ members ──membership+role──▶ every module
                                │
                                ├──▶ workspace: ownerId sync (transfer transaction)
                                ├──▶ projects: auto-transfer owned projects on
                                │     remove/leave (via projects service, same txn)
                                └──▶ notifications/email: invitation emails (Resend),
                                      invite-accepted notices (post-MVP)
```

| Contract | Detail |
|---|---|
| **workspace.ownerId** | Transfer = one transaction updating `Workspace.ownerId` + both memberships (workspace data model §4). Members service is the sole writer of this invariant |
| **projects.ownerId** | On remove/leave: `projectsService.transferOwnedProjects(userId → workspaceOwnerId)` inside the same transaction — all non-deleted projects, archived included |
| **auth** | Invitation acceptance requires `emailVerified == true`; the auth module's session context supplies `userId` for every membership check |
| **every module** | The `requireWorkspaceMember`/`requireRole` guards (auth lifecycle §7 / workspace lifecycle §5) are the enforcement point — members defines the data they read |

---

## 8. Trust Boundaries & Security Properties

1. Every membership action passes role guards: `requireRole(OWNER)` / `requireRole(OWNER | ADMIN)` — the matrix is server-enforced, never UI-gated.
2. Invitation tokens are capability tokens: single-use, expiring, revocable — possession plus acceptance creates access, nothing else.
3. Invitation acceptance is gated on a verified account — no unverified identity can enter a workspace.
4. Transfer/remove/leave are **atomic transactions** — a failed transfer leaves both roles unchanged; a failed removal leaves membership + project ownership unchanged (PRD edge cases).
5. The project-transfer recipient is fixed by rule (Workspace Owner), not chosen by the actor — no privilege escalation path via removal.
6. `404`-style responses where existence would leak (invitation status is only revealed to the invitee/inviter).

---

## 9. Non-Goals (MVP)

Per PRD §5.3 future enhancements: custom roles, permission editor, team groups, bulk invitations, organization-wide management. Also excluded: SSO/SCIM, audit-log UI (post-MVP), email notifications for invites beyond the invitation email itself.

---

## 10. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Invitation expiry period | PRD: "defined period" — pick 7 days default (configurable) |
| 2 | Can a member re-invite a previously removed user? | Likely yes (new invitation) — confirm |
| 3 | Decline vs ignore | Decline is explicit; expired covers the rest — confirm no "decline" UI needed |
| 4 | Member count limits per workspace | PRD: none — unlimited (free tier note: no quota) |
