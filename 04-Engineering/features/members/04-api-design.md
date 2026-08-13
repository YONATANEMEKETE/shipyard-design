# Members — API Design

**Module:** `apps/api/src/modules/members`
**Status:** Draft v0.1 — 2026-08-12
**Base URL:** `/api/v1` (proxied by Next.js; API internal-only)

---

## 1. Conventions

- **Resources:** `members` (workspace-scoped: `/workspaces/{workspaceId}/members/*`) and `invitations` (mixed: workspace-scoped create/list, token-based accept).
- **Guards:** every endpoint requires `requireSession`; workspace-scoped ones add `requireWorkspaceMember` + role guards per the PRD matrix. Accept/decline are keyed by the **invitation token**, not the workspace.
- **Invitation identity:** routes use the invitation **id** for inviter/Owner actions; the invitee acts through the **token** (hashed at rest).
- **Error envelope + codes:** global shape `{ error: { code, message, details? } }`, members-specific codes in §4.
- **Status codes:** `200` · `201` (invite) · `204` (remove/leave/revoke/decline) · `400` · `401` · `403` · `404` · `409` · `429`.

---

## 2. Endpoint Map

| Method | Path | Domain op | Guard |
|---|---|---|---|
| POST | `/workspaces/{wsId}/invitations` | inviteMember | OWNER (Member/Admin) · ADMIN (Member only) |
| GET | `/workspaces/{wsId}/invitations` | list invitations (pending) | OWNER / ADMIN / inviter |
| POST | `/invitations/{id}/resend` | resendInvitation | inviter / OWNER |
| POST | `/invitations/{id}/revoke` | revokeInvitation | inviter / OWNER |
| POST | `/invitations/accept` | acceptInvitation | verified session + token |
| POST | `/invitations/{id}/decline` | declineInvitation | invitee (email match) |
| GET | `/workspaces/{wsId}/members` | listMembers | any member |
| GET | `/workspaces/{wsId}/members/{userId}` | getMember | any member |
| PATCH | `/workspaces/{wsId}/members/{userId}/role` | changeRole | **OWNER** |
| DELETE | `/workspaces/{wsId}/members/{userId}` | removeMember | OWNER (Member/Admin) · ADMIN (Member) |
| POST | `/workspaces/{wsId}/members/leave` | leaveWorkspace | Member / Admin |
| POST | `/workspaces/{wsId}/transfer-ownership` | transferOwnership | **OWNER** |

---

## 3. Endpoint Details

### 3.1 Invite — `POST /workspaces/{wsId}/invitations`

**Body:** `{ email: string, role: Role }`
**Validation (Zod):** normalized email · `role ∈ { MEMBER, ADMIN }` — **OWNER rejected** (400 `MEMBER_ROLE_INVALID`).

**Guards:** role-limited: Owner may invite as MEMBER or ADMIN; Admin inviting as ADMIN → 403 `MEMBER_FORBIDDEN`.

**Business checks (in order):**
1. Email not already a member → 409 `MEMBER_ALREADY_EXISTS`
2. No existing PENDING invite for (workspace, email) → 409 `INVITATION_ALREADY_PENDING` (the DB partial index is the backstop — a Prisma P2002 unique violation maps to this same code)
3. Email ≠ inviter's own → 400 `MEMBER_CANNOT_INVITE_SELF`

**Responses:** `201` `{ invitation: { id, email, role, status: "PENDING", expiresAt } }` (token never returned — it travels only in the email link) · `400` · `403` · `409` · `429`.

### 3.2 Invite list — `GET /workspaces/{wsId}/invitations`

**Responses:** `200` `{ invitations: [{ id, email, role, status, invitedAt, expiresAt, invitedBy: { id, name } }] }` — PENDING first, then terminal. Visible to Owners/Admins and each invite's inviter.

### 3.3 Resend / Revoke — `POST /invitations/{id}/resend` · `POST /invitations/{id}/revoke`

**Guards:** inviter or **OWNER**; invitation must be PENDING (409 `INVITATION_NOT_PENDING`).

**Responses:** resend → `200` `{ invitation }` (new token hash + refreshed `expiresAt`, same row) · revoke → `204`. Unknown id → `404` (same response whether or not it exists).

### 3.4 Accept — `POST /invitations/accept`

**Body:** `{ token: string }` — the capability from the invite link.

**Guards:** `requireSession` + verified email.

**Business checks (in order):**
1. Token hash lookup → `404 INVITATION_INVALID` (unknown/expired/revoked/declined/used — one generic code)
2. **Email match:** acceptor's verified account email MUST equal `invitation.email` → `403 INVITATION_EMAIL_MISMATCH` (a forwarded link cannot join with a different account)
3. Already a member → 409 `MEMBER_ALREADY_EXISTS`

**Accept transaction (optimistic, race-safe):**
```
conditional UPDATE Invitation SET status='ACCEPTED', acceptedAt=now
  WHERE id = $1 AND status = 'PENDING'
  → 0 rows affected → 409 INVITATION_ALREADY_USED (concurrent accept lost)
INSERT Membership (workspaceId, userId, role = invitation.role)
```
Both in one Prisma transaction. **Responses:** `200` `{ workspace: { id, name } }` → web redirects to its dashboard · `400` · `403` · `404` · `409` · `429`.

### 3.5 Decline — `POST /invitations/{id}/decline`

**Guards:** session whose verified email matches the invitation's email (no token needed — the invitee is known by email; id is non-guessable cuid).

**Responses:** `204` · `404 INVITATION_INVALID` · `409 INVITATION_NOT_PENDING`.

### 3.6 List / Get member — `GET /workspaces/{wsId}/members[/{userId}]`

**Responses:** `200` `{ members: [{ id, name, email, role, joinedAt }] }` · single: `{ member: { …, role, isOwner, projectsOwned: number } }` — `projectsOwned` powers the removal dialog ("N projects will transfer to the Workspace Owner").

### 3.7 Change role — `PATCH /workspaces/{wsId}/members/{userId}/role`

**Body:** `{ role: MEMBER | ADMIN }` — OWNER not selectable (400 `MEMBER_ROLE_INVALID`); the Owner row is never a target (404 for Owner userId).

**Responses:** `200` `{ member }` (takes effect immediately — next request sees the new role) · `400` · `403` · `404`.

### 3.8 Remove — `DELETE /workspaces/{wsId}/members/{userId}`

**Guards:** Owner → target MEMBER or ADMIN; Admin → target MEMBER only (403 otherwise). Self-removal is not allowed here (400 `MEMBER_CANNOT_REMOVE_SELF` → web routes to leave flow).

**Transaction:** transfer owned projects → Workspace Owner (via projects service) + DELETE membership. **Responses:** `204` · `400` · `403` · `404`.

### 3.9 Leave — `POST /workspaces/{wsId}/members/leave`

**Guards:** Member / Admin. **The Owner cannot leave** (409 `MEMBER_LAST_OWNER` — transfer ownership first; the web explains this).

**Transaction:** same as remove (project transfer + membership delete), self-initiated. **Responses:** `204` · `409`.

### 3.10 Transfer ownership — `POST /workspaces/{wsId}/transfer-ownership`

**Body:** `{ userId: string }` — any current MEMBER or ADMIN (not self, not Owner row).

**Transaction (single):**
```
1. target still a member?  → no → 404 MEMBER_NOT_FOUND (PRD edge case:
   "target is no longer a workspace member")
2. UPDATE Workspace.ownerId = target
3. UPDATE target Membership.role = OWNER
4. UPDATE transferor Membership.role = ADMIN
```
All-or-nothing; the partial unique index guarantees exactly one OWNER row afterwards.

**Responses:** `200` `{ member }` (new owner) · `400` · `403` · `404`.

---

## 4. Error Codes (members domain)

| Code | Status | Meaning |
|---|---|---|
| `MEMBER_INVALID_INPUT` | 400 | Zod validation failed |
| `MEMBER_ROLE_INVALID` | 400 | Role not assignable (OWNER via invite/role change) |
| `MEMBER_CANNOT_INVITE_SELF` | 400 | Self-invitation |
| `MEMBER_CANNOT_REMOVE_SELF` | 400 | Self-removal via remove endpoint |
| `MEMBER_UNAUTHORIZED` | 401 | No valid session |
| `MEMBER_FORBIDDEN` | 403 | Role insufficient (Admin inviting Admin, Admin removing Admin, etc.) |
| `INVITATION_EMAIL_MISMATCH` | 403 | Acceptor's email ≠ invited email |
| `MEMBER_NOT_FOUND` | 404 | Unknown member / transfer target gone |
| `INVITATION_INVALID` | 404 | Token unknown, expired, revoked, declined, or used (one generic code) |
| `MEMBER_ALREADY_EXISTS` | 409 | Email already a member / invitee already joined |
| `INVITATION_ALREADY_PENDING` | 409 | Pending invite exists for (workspace, email) |
| `INVITATION_NOT_PENDING` | 409 | Resend/revoke/decline on a terminal invitation |
| `INVITATION_ALREADY_USED` | 409 | Concurrent accept lost the conditional update |
| `MEMBER_LAST_OWNER` | 409 | Owner tried to leave/remove self without transfer |
| `MEMBER_RATE_LIMITED` | 429 | Rate limit hit |

---

## 5. Rate Limiting

| Endpoint | Limit |
|---|---|
| `POST /workspaces/{wsId}/invitations` | 10/min per user |
| `resend` / `revoke` / `decline` | 10/min per user |
| `POST /invitations/accept` | 10/min per user |
| `PATCH …/role` · `DELETE …/members/{userId}` · `transfer-ownership` · `leave` | 10/min per user |
| `GET …/members` (+ single) | 60/min per user |

---

## 6. Web Integration (consumers)

| Web surface | Endpoint(s) |
|---|---|
| Invite Members modal (Members directory) | `POST /workspaces/{wsId}/invitations` |
| Members directory + Pending Invitations tab | `GET …/members` · `GET …/invitations` · `resend` · `revoke` |
| Invitation link page (`/invite?token=…`) | `POST /invitations/accept` · `decline` |
| Member Details drawer | `GET …/members/{userId}` · `PATCH …/role` · `DELETE …/members/{userId}` |
| User menu → Leave workspace | `POST …/members/leave` |
| Workspace Settings → Members / ownership | `transfer-ownership` |
| Auth flow integration | Invite link → login/register/verify → invitation resumes automatically (auth lifecycle) |

All calls go **through the Next proxy** (ADR-003).

---

## 7. OpenAPI & Shared Contracts

- Zod schemas in `packages/shared`: `InviteMemberInput`, `AcceptInvitationInput`, `ChangeRoleInput`, `TransferOwnershipInput`, `MemberResponse`, `InvitationResponse` — shared with web + OpenAPI generation.
- `Role` enum is defined in `packages/shared` (imported by Prisma-facing code and the web) — one source of truth.

---

## 8. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Invite email copy + expiry display | Copy lives in web; expiry shown in pending tab |
| 2 | Decline UX | Covered by the invite page — confirm wording during implementation |
| 3 | Member directory pagination | Teams ≤ 30 per target segment — no pagination in MVP (revisit at scale) |
