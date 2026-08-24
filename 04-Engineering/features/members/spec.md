# Members — Feature Spec

**Status:** Approved
**Last updated:** 2026-08-22
**Design sources:** PRD §5.3 · §6 · §7.2 · UX User Flow 7 (invitations)
**Technical design:** Excluded by design — produced during this feature's implementation step, driven by this behavioral spec.

---

## 1. What this feature is about

Members owns **who can cross a workspace's boundary, with what role**: memberships, the Owner/Admin/Member role model, invitations, and membership lifecycle (join, remove, leave, role change, ownership transfer). It also owns the rule that projects owned by a departing member transfer to the Workspace Owner.

## 2. What users can do

- View the member directory (name, email, role, join date).
- Invite people by email (role-limited); resend, revoke, and see invitation status.
- Accept or decline a pending invitation (verified account required).
- Change a member's role (Owner only; Member ⇄ Admin).
- Remove members (role-limited).
- Leave a workspace on their own.
- Transfer workspace ownership (Owner only).
- Open the owner's "cage": the Owner cannot leave, be removed, or be demoted — only transfer, which demotes them to Admin.

## 3. Main behaviors & actions

### 3.1 Roles (permission matrix — PRD §6)

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

### 3.2 Invitations
- A pending invitation grants **no access** — accepting is the only door.
- Owner invites as Member or Admin; Admin invites as **Member only**; the Owner role is never offered via invitation.
- Cannot invite: yourself, an existing member, an email already in the workspace, or an email that already has a pending invitation (resend instead).
- Invitations expire; can be revoked before acceptance; cannot be accepted twice.
- Accept paths: existing verified user accepts directly; new user registers + verifies first, then the invitation is resumed automatically.
- Acceptance always requires a verified account.

### 3.3 Membership lifecycle
- One membership per user per workspace, with exactly one role; roles are independent per workspace.
- Role changes and removals take effect immediately (no permission caching).
- **Remove:** Owner removes Members/Admins; Admin removes Members only. Access is gone immediately.
- **Leave:** any member or admin may leave; the Owner must transfer ownership first.
- **Ownership transfer (Owner only):** recipient becomes Owner, transferor becomes Admin — the transfer either completes entirely or not at all.
- When a member is removed or leaves, **all projects they own transfer automatically to the Workspace Owner** in the same operation — archived projects included. The recipient is fixed by rule (workspace Owner), never chosen by the actor.

## 4. User flows (high level)

1. **Invite:** settings → members → invite email + role → email link → accept → verified user becomes member with offered role.
2. **New user invite:** link → register + verify → invitation resumed → accept → member.
3. **Remove:** member directory → remove → confirm → access revoked immediately + owned projects transferred.
4. **Leave:** member → settings → leave → confirm → farewell; owned projects transferred.
5. **Transfer ownership:** Owner → transfer to member → recipient becomes Owner, transferor becomes Admin (one operation).
6. **Invitation management:** resend / revoke from the pending list.

## 5. Business rules

1. Every workspace has exactly one Owner membership.
2. Role changes and removals take effect immediately.
3. A removed member loses access immediately; a pending invitation grants nothing.
4. The Owner role is never granted by invitation; Owners invite as Member/Admin, Admins as Member only.
5. Admins remove Members only; the Owner changes roles and transfers ownership.
6. The Owner cannot leave or be removed before transferring ownership (transfer is a single operation).
7. Owned projects transfer to the Workspace Owner on remove/leave, archived included — same operation, fixed recipient.
8. Invitations expire, can be revoked, and cannot be accepted twice.
9. Invitation acceptance requires a verified account.
10. Transfer/remove/leave are all-or-nothing: a failed operation changes nothing.

## 6. Out of scope (MVP)

Custom roles, permission editor, team groups, bulk invitations, org-wide management, SSO/SCIM, audit-log UI.

## 7. Open product questions

| # | Question | Notes |
|---|---|---|
| 1 | Invitation expiry period | 7 days default, configurable |
| 2 | Re-invite a previously removed user | Likely yes (new invitation) — confirm |
| 3 | Decline vs ignore | Explicit decline covers the rest via expiry — confirm no separate "ignore" state |
| 4 | Member count limits | None in PRD |
