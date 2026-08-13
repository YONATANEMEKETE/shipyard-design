# Members — Request Lifecycle

**Module:** `apps/api/src/modules/members`
**Status:** Draft v0.1 — 2026-08-12
**Relies on:** workspace lifecycle §5 (membership hot path — not repeated here) · `02-data-model.md` · `04-api-design.md`

---

## 1. Overview

Members traffic splits into two families:

1. **Workspace-scoped actions** (invite, list, role, remove, leave, transfer) — ride the standard `requireSession → requireWorkspaceMember → requireRole(...)` chain (workspace lifecycle §5) plus members-specific guards.
2. **Invitation flows** (accept/decline) — the interesting ones: keyed by **token** (accept) or email (decline), gated by verified identity, and race-safe by design.

The two flows that demand precision: **accept** (capability token + email match + conditional update) and **transfer** (three-write atomic swap). Everything else is a guarded row mutation.

---

## 2. Flow — Invite (workspace-scoped)

```
1. POST /workspaces/{wsId}/invitations
   body: { email, role }                        [Zod: email normalized, role MEMBER|ADMIN]
2. requireSession → requireWorkspaceMember
3. role guard: ADMIN may only invite MEMBER     [else 403 MEMBER_FORBIDDEN]
4. business checks:
   ├── email already a member        → 409 MEMBER_ALREADY_EXISTS
   ├── pending invite exists         → 409 INVITATION_ALREADY_PENDING
   │     (DB partial unique index is the backstop: a P2002 violation
   │      maps to this same code — race-proof by Postgres)
   └── email == inviter's            → 400 MEMBER_CANNOT_INVITE_SELF
5. INSERT Invitation (PENDING, role, token HASHED at rest, +7d expiry)
6. Resend email via Resend → invite link /invite?token=... (plaintext token
   appears ONLY here — never in API responses, never in the DB)
7. 201 → { invitation } (no token) → pending tab updates
```

---

## 3. Flow — Accept invitation (token-based, race-safe)

The invitee's journey: link → (auth gate) → accept transaction → workspace.

```
1. Email arrives → click → web /invite?token=T
2. WEB AUTH GATE (auth module):
   ├── not signed in      → /login (or /register) — after verification,
   │                        the invitation RESUMES automatically (PRD flow 7)
   └── signed in          → continue
3. POST /invitations/accept { token }
4. requireSession → verified email (auth gate)
5. lookup by token hash  → unknown/expired/revoked/declined/used
                            → 404 INVITATION_INVALID (one generic code)
   ├── expired check: PENDING + expiresAt < now → same 404
   └── EMAIL MATCH: session email MUST equal invitation.email
                            → 403 INVITATION_EMAIL_MISMATCH
                            (a forwarded link can't join with another account)
6. ACCEPT TRANSACTION (single, optimistic):
   a. conditional UPDATE Invitation SET status='ACCEPTED', acceptedAt=now
        WHERE id = $1 AND status = 'PENDING'
        → 0 rows affected → 409 INVITATION_ALREADY_USED
          (concurrent accept lost — loser gets the friendly conflict)
   b. INSERT Membership (workspaceId, userId, role = invitation.role)
   — both or neither
7. 200 → { workspace: { id, name } } → web redirects to its dashboard
   (switcher + auto-enter logic takes over from here)
```

**Race matrix:** two simultaneous accepts → one wins the conditional update, the other gets 409 (idempotent outcome: exactly one membership). Accept vs revoke → whichever transaction commits first; the loser's conditional update finds status ≠ PENDING.

---

## 4. Flow — Decline / Resend / Revoke (small mutations)

```
DECLINE (invitee):
  POST /invitations/{id}/decline
    → session email must match invitation.email    [404 INVITATION_INVALID
      (id is a non-guessable cuid; mismatch answers 404 — no existence leak)]
    → must be PENDING                              [409 INVITATION_NOT_PENDING]
    → UPDATE status = DECLINED, declinedAt = now   [204]

RESEND (inviter / OWNER):
  POST /invitations/{id}/resend
    → must be PENDING                              [409]
    → UPDATE token (new hash) + expiresAt (fresh 7d) — same row [200]

REVOKE (inviter / OWNER):
  POST /invitations/{id}/revoke
    → must be PENDING                              [409]
    → UPDATE status = REVOKED, revokedAt = now     [204]
    → the old link is now dead (404 on accept)
```

## 5. Flow — Remove member (with project auto-transfer)

```
1. DELETE /workspaces/{wsId}/members/{userId}
2. requireSession → requireWorkspaceMember
3. role guard (PRD matrix):
   ├── OWNER → target MEMBER or ADMIN allowed
   ├── ADMIN → target MEMBER only            [else 403 MEMBER_FORBIDDEN]
   └── target == self → 400 MEMBER_CANNOT_REMOVE_SELF (web routes to leave)
4. REMOVAL TRANSACTION (single):
   a. projectsService.transferOwnedProjects(userId → ws.ownerId)
      — ALL owned projects, archived included (domain rule 9)
      — recipient is fixed by rule, not chosen (no escalation path)
   b. DELETE Membership (workspaceId, userId)
   — both or neither; failure leaves membership + ownership unchanged
5. 204 → removed member loses access on their VERY NEXT request
   (per-request membership check — no cached permissions)
   → directory updates for everyone else
```

## 6. Flow — Leave workspace (self-removal)

```
1. POST /workspaces/{wsId}/members/leave
2. requireSession → requireWorkspaceMember
3. role check:
   ├── MEMBER / ADMIN → proceed
   └── OWNER → 409 MEMBER_LAST_OWNER
        (web explains: transfer ownership first — the Owner's cage)
4. Same transaction as remove (project transfer + membership delete)
5. 204 → web redirects to another workspace / onboarding
```

## 7. Flow — Transfer ownership (three-write atomic swap)

```
1. POST /workspaces/{wsId}/transfer-ownership
   body: { userId }                            [Zod: cuid]
2. requireSession → requireWorkspaceMember → requireRole(OWNER)
3. target checks:
   ├── target ≠ self, target ≠ Owner row       [400 MEMBER_ROLE_INVALID / 404]
   └── target is a CURRENT member              [404 MEMBER_NOT_FOUND —
        PRD edge case: "target is no longer a workspace member"
        (checked INSIDE the transaction to close the race)]
4. TRANSFER TRANSACTION (single):
   a. re-verify target membership (inside txn — no TOCTOU)
   b. UPDATE Workspace.ownerId = target
   c. UPDATE target Membership.role = OWNER
   d. UPDATE transferor Membership.role = ADMIN
   — the partial unique index guarantees exactly one OWNER row after commit;
     if the invariant would break, Postgres rejects the whole transaction
5. 200 → { member } (new owner)
   → badges update on everyone's next request; transferor sees Admin surfaces
```

## 8. Flow — Change role (Owner only)

```
1. PATCH /workspaces/{wsId}/members/{userId}/role
   body: { role: MEMBER | ADMIN }              [Zod; OWNER rejected 400]
2. guards: requireRole(OWNER); target is a Member/Admin row (404 for Owner row)
3. UPDATE Membership.role                        [200 { member }]
4. Effect is immediate: next request from that user carries the new role
   (guards read the row per request — no session/role cache)
```

## 9. Edge Cases & Failure Handling

| Case | Behavior |
|---|---|
| Two users accept the same invite concurrently | Conditional update → one wins, loser gets 409 `INVITATION_ALREADY_USED`; exactly one membership exists |
| Accept vs revoke race | Same conditional update — the loser finds status ≠ PENDING |
| Forwarded invite link + different account | 403 `INVITATION_EMAIL_MISMATCH` — link is tied to the invited email |
| Invitee never accepts | Invitation expires silently (derived from `expiresAt`); pending tab shows it; re-invite legal |
| Owner tries to leave/remove self | 409 `MEMBER_LAST_OWNER` — transfer first, then leave as Admin |
| Transfer target leaves mid-flight | Membership re-verified inside the transaction → 404, nothing changes |
| Removed member uses an old link/URL | 403 on the very next request (workspace hot path) |
| Admin tries to remove an Admin / invite an Admin | 403 `MEMBER_FORBIDDEN` (matrix enforced server-side) |
| Remove/leave transaction fails mid-way | Rollback — membership AND project ownership unchanged (PRD) |
| Invite to an email that later registers | Accept resumes after verification — the invitation row was waiting all along |
| Email send fails (Resend) | Invitation exists but undelivered → resend action repairs it (no duplicate row) |

## 10. Dev vs Prod Differences

| Concern | Local dev | Production |
|---|---|---|
| Invite/reset emails | Resend test mode / mailpit-style capture | Resend prod + verified domain (DKIM/SPF) |
| Invite link origin | `http://localhost:3000/invite?token=…` | `https://shipyard.yonatanem.com/invite?token=…` |
| Rate limits | Same (catches abuse early) | Same |
| Partial unique indexes | Same migrations (Postgres both sides) | Same |

