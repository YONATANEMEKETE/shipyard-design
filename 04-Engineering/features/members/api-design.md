# Members — API Design

**Status:** Approved for implementation (F3)
**Last updated:** 2026-08-30
**Sources:** `features/members/spec.md` · `features/members/data-model.md` (locked) · `features/workspace/spec.md` · `features/workspace/data-model.md` + `features/workspace/api-design.md` (F2 precedent — `:slug` context, guard chain, archived matrix, error envelope) · `features/auth/data-model.md` + `features/auth/api-design.md` (F1 precedent — `user.emailVerified` gate, Better Auth session) · `00-architecture.md` §5–§8 · `ADR-001` (stack) · `ADR-002` (shared contracts) · `ADR-003` (Next → API proxy) · `Implementation Plan.md` F3

> **Principle:** Like workspace (F2), **every route in this module is hand-written Shipyard code**, following the canonical pipeline:
>
> ```text
> route → validation → permission check → controller → service → repository → Prisma
> ```
>
> Better Auth handles identity only (who you are, `emailVerified`). This module owns **authorization** — who may cross a workspace boundary, with what role, and through which invitation door — end to end. It establishes the RBAC pattern that F4–F11 copy.

---

## 1. Base path & conventions

| Concern | Choice |
|---|---|
| Base path (workspace-scoped) | `/api/v1/workspaces/:slug/members` and `/api/v1/workspaces/:slug/invitations` — mirrors `features/workspace/api-design.md` §1; `:slug` is the same immutable short token, disambiguates duplicate names; inter-table references stay on `id` |
| Base path (token-gated) | `/api/v1/invitations/:token` — global, not workspace-scoped; the token is the only key; preview/accept/decline live here so a non-member can reach them without a workspace context |
| Next.js proxy | Browser never hits the API directly (ADR-003); `apps/web` forwards `/api/v1/*` → `http://api:4000/api/v1/*`, cookies forwarded; Caddy exposes only `web:3000` |
| Auth transport | HttpOnly Better Auth session cookie read by `requireSession` from F1; nothing new — `req.session.userId` + `user.emailVerified` are the only identity inputs |
| Validation | Zod schemas from `packages/shared` (`data-model.md` §4) at the route boundary; API rejects anything the client UI would never send |
| Envelope | Success: resource JSON directly (or `{ members: [...] }` / `{ invitations: [...] }` for collections). Failure: `{ "error": { "code": "...", "message": "...", "details"? } }` matching the F1/F2 global error handler |
| Workspace context | Reuses the F2 shared middleware `resolveWorkspaceContext(:slug)` verbatim — one authoritative resolution per request, leak-free `404 WORKSPACE_NOT_FOUND` when slug unknown or caller not a member (data-model §6.1) |
| Archived enforcement | Reuses `resolveWorkspaceContext({ rejectArchived: true })` for every mutating workspace-scoped route; `GET` routes pass `rejectArchived: false` so the directory remains readable in an archived workspace |

---

## 2. Endpoint inventory

Thirteen endpoints cover every behavior in `spec.md` §2–§5 and `data-model.md` §6. No extras. Grouped by scope so the guard chain is obvious.

### 2.1 Workspace-scoped — directory & membership lifecycle

All under `/api/v1/workspaces/:slug/...`, all go through the §4 guard chain.

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 1 | `GET` | `/api/v1/workspaces/:slug/members` | `requireSession` → member (any role) | §2 View member directory (name, email, role, join date). Allowed while `ARCHIVED` (read-only). |
| 2 | `GET` | `/api/v1/workspaces/:slug/members/:memberId` | `requireSession` → member (any) | Member Details drawer data — same payload as #1 for one row. Convenience for direct fetch; workspace-scoped so `memberId` is validated to belong to `:slug`. |
| 3 | `PATCH` | `/api/v1/workspaces/:slug/members/:memberId/role` | `requireSession` → member → `OWNER` + `rejectArchived` | §3.1/§3.3 Change Member ⇄ Admin (Owner only). Body `{ role: "MEMBER" \| "ADMIN" }`. |
| 4 | `POST` | `/api/v1/workspaces/:slug/members/:memberId/remove` | `requireSession` → member → `OWNER\|ADMIN` (narrowed in service) + `rejectArchived` | §3.3 Remove member — role-limited (Owner removes Member/Admin; Admin removes Member only). Body `{ confirm: true }`. Shows project-transfer count in response. |
| 5 | `POST` | `/api/v1/workspaces/:slug/leave` | `requireSession` → member (`ADMIN\|MEMBER` only, Owner rejected in service) + `rejectArchived` | §3.3 Leave workspace on own. Body `{ confirm: true }`. Owner must transfer first. |
| 6 | `POST` | `/api/v1/workspaces/:slug/transfer-ownership` | `requireSession` → member → `OWNER` + `rejectArchived` | §3.3 Transfer workspace ownership — recipient becomes Owner, caller becomes Admin, one atomic operation. Body `{ targetMemberId }`. |

### 2.2 Workspace-scoped — invitations (management)

Also under `/api/v1/workspaces/:slug/...`, same guard-chain family.

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 7 | `POST` | `/api/v1/workspaces/:slug/invitations` | `requireSession` → member → `OWNER\|ADMIN` (service narrows role) + `rejectArchived` | §3.2 Invite by email — Owner invites as Member/Admin, Admin as Member only. Body `{ emails: string[], role: "MEMBER"\|"ADMIN" }`. Batch ≤20, transactional. |
| 8 | `GET` | `/api/v1/workspaces/:slug/invitations` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived: false` | §4.6 Invitation management — list invitations for the workspace (all statuses, `PENDING` first, then `expiresAt` desc). Allowed while archived for audit. Member role receives `403` — they never see the invite list. |
| 9 | `POST` | `/api/v1/workspaces/:slug/invitations/:invitationId/resend` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | §4.6 Resend — re-sends same `token` link, bumps `updatedAt`. Body `{ confirm: true }`. Rate-limited. |
| 10 | `POST` | `/api/v1/workspaces/:slug/invitations/:invitationId/revoke` | `requireSession` → member → `OWNER\|ADMIN` + `rejectArchived` | §3.2 Revoke before acceptance. Body `{ confirm: true }`. `PENDING → REVOKED`. |

### 2.3 Token-gated — invitation acceptance (not workspace-scoped)

Under `/api/v1/invitations/:token/...` — the caller is authenticated but **not** a workspace member (the token is the door). No `resolveWorkspaceContext` here; the service resolves the workspace from the `token` row.

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 11 | `GET` | `/api/v1/invitations/:token` | `requireSession` (verified required in service) | §4.1/§4.2 Invitation preview — shows `workspaceName`, `workspaceIcon`, `role`, `email`, `expiresAt`, `status` so the UI can render accept/decline or the expired/revoked message. No workspace membership required. Rejected while workspace `ARCHIVED` (spec archived matrix). |
| 12 | `POST` | `/api/v1/invitations/:token/accept` | `requireSession` (verified required) | §3.2 Accept — creates `workspace_member` in same transaction, marks `ACCEPTED`. Verified account required. Idempotent race-safe. |
| 13 | `POST` | `/api/v1/invitations/:token/decline` | `requireSession` (verified required) | §7 Q3 decline — `PENDING → DECLINED`. Verified account required. |

> **Why two families:** Workspace-scoped routes answer "what can I do *inside* this workspace?" Token-gated routes answer "can I *enter* this workspace via this link?" The token is the only key for the second family — putting accept under `/workspaces/:slug/...` would require the caller to already be a member, defeating the purpose.

---

## 3. Context resolution

### 3.1 Workspace-scoped routes (#1–#10) — reuse F2's resolver

The single lookup used by every workspace-scoped endpoint is **exactly** the F2 `resolveWorkspaceContext(:slug)` middleware (api-design.md §3):

```text
req.params.slug  ──findFirst──▶  workspace (by slug)  +  membership of req.session.userId on that workspace
```

- **One query** joins both; then the guard chain evaluates:
  - No workspace with that slug **OR** no membership row ⇒ identical generic `404 WORKSPACE_NOT_FOUND` — a non-member and a bogus slug are indistinguishable (no existence leak).
  - Membership exists but role insufficient for the route (checked by `requireWorkspaceRole` or finer service logic for #4) ⇒ `403 FORBIDDEN_ROLE`.
  - Membership exists, role OK, workspace `ARCHIVED`, route has `rejectArchived: true` ⇒ `409 WORKSPACE_ARCHIVED` (workspace `api-design.md` §6).
- Resolved context is attached once (`req.workspaceContext = { workspaceId, slug, status, role, memberId, userId }`) so controller/service never re-resolve — one authoritative resolution per request.
- Implemented as Express middleware in `apps/api/src/common/guards/workspace-context.ts` (already shipped in F2); members just imports it.

### 3.2 Token-gated routes (#11–#13) — invitation lookup

A different single lookup, owned by this module:

```text
req.params.token  ──findUnique──▶  invitation (by token)  +  workspace (via invitation.workspaceId)
                                   +  accepting user (via req.session.userId → user row for emailVerified)
```

- Unknown `token` ⇒ `404 INVITATION_NOT_FOUND` (generic — a random token and a revoked token are indistinguishable at the boundary only if desired; for UX we distinguish `REVOKED`/`EXPIRED`/`ACCEPTED` via the preview payload, but a completely unknown token is still 404).
- Known token but `status !== PENDING` or `expiresAt <= now()` ⇒ still `200` for preview (#11) with the status so the UI can explain; `409 INVITATION_NOT_USABLE` (with `details.status`) for accept/decline (#12–#13).
- Known token, workspace `ARCHIVED` ⇒ `409 WORKSPACE_ARCHIVED` — cannot join a frozen container (data-model §6.3).
- `user.emailVerified === false` ⇒ `403 EMAIL_NOT_VERIFIED` — must verify before accepting (spec §3.2, data-model §5).
- No `resolveWorkspaceContext` — the workspace is derived from the invitation row, not the URL.

---

## 4. Guard chain (canonical, per Implementation Plan §1.4)

### 4.1 Workspace-scoped chain (#1–#10)

```text
requireSession                     ← F1 middleware: valid Better Auth session else 401
  │                                 also loads user.emailVerified for #12–#13
  │
resolveWorkspaceContext(slug)      ← F2 shared middleware; §3.1
  │                                  404 generic on miss/non-membership (no leak)
  │                                  409 WORKSPACE_ARCHIVED when rejectArchived && ARCHIVED
  ├─ requireWorkspaceRole(...)     ← only where the matrix allows a simple set:
  │                                  #3 OWNER, #6 OWNER, #8 OWNER|ADMIN
  │                                  (#4 needs finer service logic — Owner vs Admin target)
  │
controller → service               ← service revalidates permission matrix + state
                                     preconditions (e.g., Admin cannot invite Admin,
                                     cannot remove Owner, Owner cannot leave) inside
                                     the same transaction as writes — defense in depth
```

Rules this reaffirms for all future milestones:

1. The URL carries workspace context; there is no hidden server-side "active workspace" state.
2. Membership resolution happens exactly once per request, generically, with leak-free responses.
3. Role/state checks live in named guards or service preconditions — never inline ad-hoc queries inside controllers.
4. Archived workspaces are read-only — mutating routes use `rejectArchived: true` at the guard layer and reassert in the service.

### 4.2 Token-gated chain (#11–#13)

```text
requireSession                     ← 401 if no session
  │
invitationByToken lookup           ← 404 if unknown token
  │
service preconditions              ← 403 EMAIL_NOT_VERIFIED, 409 WORKSPACE_ARCHIVED,
                                     409 INVITATION_NOT_USABLE / INVITATION_EXPIRED,
                                     409 ALREADY_MEMBER (if user already in workspace)
                                     — checked inside the accepting transaction for #12
  │
controller → service → repository  ← #12's membership insert + status flip are one Prisma $transaction
```

---

## 5. Request/response contracts

All schemas from `packages/shared` (`data-model.md` §4) plus the two member-specific schemas below. Route handlers validate bodies/params before anything else.

### 5.1 Workspace-scoped

| Endpoint | Body | Success response |
|---|---|---|
| #1 list members | — | `200` + `{ members: workspaceMemberCardSchema[] }` sorted `role` (Owner first) then `createdAt` asc |
| #2 get member | — | `200` + `workspaceMemberCardSchema` |
| #3 change role | `changeMemberRoleSchema` `{ memberId, role: "MEMBER"\|"ADMIN" }` | `200` + `workspaceMemberCardSchema` (updated) |
| #4 remove | `{ confirm: true }` (literal) + `:memberId` param | `200` + `{ removedMemberId, transferredProjects: number }` — count from `transferOwnedProjects` (data-model §7); `0` until F4 ships the implementation (Checkpoint A) |
| #5 leave | `{ confirm: true }` | `200` + `{ transferredProjects: number }` or `204` empty — 200 with count is friendlier for the confirmation dialog |
| #6 transfer | `transferOwnershipSchema` `{ targetMemberId }` | `200` + `{ members: workspaceMemberCardSchema[] }` (both updated rows) or `200` + updated target card — either is fine; spec requires both roles have flipped |
| #7 invite | `inviteMembersSchema` `{ emails: string[1..20], role }` | `201` + `{ invitations: invitationCardSchema[] }` — one row per email, each with its `token` (privileged callers only) |
| #8 list invitations | — (query `?status=PENDING` optional) | `200` + `{ invitations: invitationCardSchema[] }` — includes `token` for Owner/Admin so resend can be issued; never cached |
| #9 resend | `{ confirm: true }` | `200` + `invitationCardSchema` (updated `updatedAt`) |
| #10 revoke | `{ confirm: true }` | `200` + `invitationCardSchema` (`REVOKED`) |

Validation details:
- `#7` emails are lowercased/trimmed by Zod, then re-validated in service: not self, not existing member email, not already `PENDING` — each maps to a distinct `409` code so the UI can explain (see §7). The whole batch is transactional — if any email fails the pre-check, none are inserted.
- `#3` rejects `role === "OWNER"` at Zod layer (not in the enum), and service rejects targeting the Owner or self.
- `#4`/`#5`/`#6`/`#9`/`#10` require the literal `confirm: true` — missing literal ⇒ `400 CONFIRMATION_REQUIRED` (same precedent as workspace archive/restore).
- `#6` `targetMemberId` must belong to the same `workspaceId` and not be the caller; service verifies liveness inside the transaction.

### 5.2 Token-gated

| Endpoint | Body | Success response |
|---|---|---|
| #11 preview | — | `200` + `invitationPreviewSchema` `{ workspaceName, workspaceIcon, role, email, expiresAt, status }` |
| #12 accept | `{}` or `{}` with optional `confirm: true` — empty body suffices; session is proof | `201` + `{ member: workspaceMemberCardSchema, workspaceSlug: string }` — `201` because a membership row is created; slug lets the client navigate into `/w/:slug` |
| #13 decline | `{}` | `200` + `invitationCardSchema` (`DECLINED`) |

Preview is `200` even when the invitation is not usable — the status field lets the client render expired/revoked/accepted without an extra call. Accept/decline on a non-`PENDING` row return `409` (see §7).

---

## 6. Archived read-only enforcement matrix

`workspace.status = ARCHIVED` flips the whole workspace read-only except lifecycle exits that live in `features/workspace` (restore/delete). For members:

| Endpoint | While ARCHIVED | Rationale (data-model §6.3) |
|---|---|---|
| #1 list members | ✅ allowed | Read-only directory remains visible |
| #2 get member | ✅ allowed | Same |
| #3 change role | ❌ `409 WORKSPACE_ARCHIVED` | No membership edits in a frozen container |
| #4 remove | ❌ `409` | Same |
| #5 leave | ❌ `409` | Frozen — membership cannot be mutated |
| #6 transfer ownership | ❌ `409` | Same — ownership is workspace state |
| #7 invite | ❌ `409` | No new doors into a frozen container |
| #8 list invitations | ✅ allowed | Audit — see pending/revoked while archived |
| #9 resend | ❌ `409` | Mutating |
| #10 revoke | ❌ `409` | Mutating (even though revoke is "undoing", it mutates) |
| #11 preview | ✅ allowed but accept will fail | Preview is read-only; accept checks `ARCHIVED` |
| #12 accept | ❌ `409 WORKSPACE_ARCHIVED` | Cannot join a frozen workspace |
| #13 decline | ❌ `409` while workspace archived | Consistent with frozen — decline is a mutation |

Enforced at the guard layer (`rejectArchived: true`) for #3–#7, #9–#10 and in service for #12–#13 (token lookup carries `workspace.status`). Defense in depth: service reasserts even when guard already ran.

---

## 7. Error codes

All errors flow through the global Express error handler with the standard envelope (§1). Services throw typed domain errors; handlers never build envelopes by hand.

| Code | HTTP | When | Notes |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | Zod body/param failure (bad email, empty batch, `role` not in `MEMBER\|ADMIN`, missing `confirm`) | `details` lists field paths |
| `CONFIRMATION_REQUIRED` | 400 | #4/#5/#6/#9/#10 without literal `confirm: true` | Same precedent as workspace |
| `WORKSPACE_NOT_FOUND` | 404 | Unknown `:slug` **or** caller isn't a member — deliberately identical | No existence leak (§3.1) |
| `MEMBER_NOT_FOUND` | 404 | `:memberId` not in this workspace | Scoped — not a cross-workspace leak |
| `INVITATION_NOT_FOUND` | 404 | Unknown `:token` (token-gated) or unknown `:invitationId` within this workspace | Token unknown is 404; `invitationId` unknown within workspace is 404 scoped |
| `FORBIDDEN_ROLE` | 403 | Member but not in allowed set for the route (e.g., Member trying #7, Admin trying to invite Admin, Admin trying #3) | Matters from F3 onward; tested with seeded roles |
| `EMAIL_NOT_VERIFIED` | 403 | Accept/decline/preview when `user.emailVerified === false` | Spec §3.2 — must verify first |
| `CANNOT_INVITE_SELF` | 409 | `#7` email equals caller's own email (case-insensitive) | Per-email code when batch contains self |
| `ALREADY_MEMBER` | 409 | `#7` email already a member of this workspace | Service check before DB |
| `PENDING_EXISTS` | 409 | `#7` email already has a `PENDING` invitation in this workspace | Resend instead |
| `CANNOT_CHANGE_OWNER_ROLE` | 409 | `#3` target is the Owner | Owner cage — only transfer changes Owner |
| `CANNOT_REMOVE_OWNER` | 409 | `#4` target is the Owner | Owner cannot be removed |
| `CANNOT_REMOVE_SELF` | 409 | `#4` caller targeting themselves | Redirect to leave flow |
| `TRANSFER_REQUIRED` | 409 | `#5` caller is Owner trying to leave | Must transfer first |
| `TRANSFER_TARGET_INVALID` | 409 | `#6` target not in workspace, is Owner, or is caller | Liveness rechecked in transaction |
| `INVITATION_NOT_USABLE` | 409 | `#12`/`#13` token row is `REVOKED`/`DECLINED`/`ACCEPTED` | `details.status` tells which |
| `INVITATION_EXPIRED` | 409 | `#12`/`#13` `expiresAt <= now()` or status `EXPIRED` | Distinct from `NOT_USABLE` for messaging |
| `ALREADY_MEMBER` (accept) | 409 | `#12` user already a member of that workspace | Race or duplicate accept |
| `WORKSPACE_ARCHIVED` | 409 | Mutating op while workspace `ARCHIVED` (§6) | Restorable via workspace restore |
| `UNAUTHENTICATED` | 401 | Missing/expired session cookie | Handled by F1 `requireSession` |
| `RATE_LIMITED` | 429 | Resend/invite rate limit (wiring finalized at F12; global limiter exists already) | `Retry-After` header |

All `409` codes are domain conflicts, not server errors — the client renders them as recoverable messages, never raw dumps. `PENDING_EXISTS` is the signal to offer "Resend instead" in the UI.

---

## 8. Sequences

### 8.1 Invite → accept (existing verified user, happy path)

```text
Owner → POST /api/v1/workspaces/:slug/invitations {emails:["bob@example.com"], role:"MEMBER"}
→ requireSession ✓ → resolveWorkspaceContext ✓ (Owner) → Zod validate
→ service: for each email (lowercased) check not self / not member / not pending
→ tx { insert invitation (token, expiresAt=now+7d) } per email → 201 invitations
→ after commit: Resend email with link /invite/:token (local dev: test adapter captures)
→ Bob (already verified) opens link → GET /api/v1/invitations/:token
→ requireSession ✓ (Bob's session) → lookup token → 200 preview { workspaceName, role, ... }
→ Bob clicks Accept → POST /api/v1/invitations/:token/accept {}
→ requireSession ✓ → service tx {
     find invitation by token FOR UPDATE
     assert status===PENDING && expiresAt>now()
     assert user.emailVerified===true
     assert no membership for (workspaceId, userId)
     insert workspace_member { role: invitation.role }
     update invitation status=ACCEPTED
   } → 201 { member, workspaceSlug }
→ client navigates to /w/:workspaceSlug (dashboard)
```

### 8.2 New user invite (register + verify → auto-resume)

```text
Invitee has no account → opens /invite/:token link
→ GET /api/v1/invitations/:token → 401 UNAUTHENTICATED (no session)
→ web redirects to /sign-up?next=/invite/:token (or /sign-in)
→ user registers + verifies (F1 flow) → session exists, emailVerified=true
→ web resumes → GET /api/v1/invitations/:token → 200 preview
→ POST /api/v1/invitations/:token/accept → same tx as §8.1 → member
```

No special API support needed — the token is just a URL the client holds across the auth flow.

### 8.3 Admin invites Member only

```text
Admin → POST /api/v1/workspaces/:slug/invitations {emails:["carol@example.com"], role:"ADMIN"}
→ resolveWorkspaceContext role=ADMIN → service checks requestedRole==="ADMIN"
→ 403 FORBIDDEN_ROLE — Admin cannot invite as Admin
→ Admin retries with role:"MEMBER" → 201 invitation (Member)
```

### 8.4 Remove member (Checkpoint A vs B)

```text
Owner/Admin → POST /api/v1/workspaces/:slug/members/:memberId/remove {confirm:true}
→ resolveWorkspaceContext + service permission check:
     Owner can remove Member/Admin; Admin can remove Member only; never Owner; never self
→ service tx {
     re-resolve current Owner userId in same tx (fixed recipient, never caller-supplied)
     call projectsService.transferOwnedProjects(workspaceId, fromUserId, toOwnerUserId, tx)
       → 0 in Checkpoint A (no projects yet, no-op)
       → N in Checkpoint B (including ARCHIVED projects)
     delete workspace_member where id=:memberId
   } → 200 { removedMemberId, transferredProjects: N }
→ removed user next request → 404 WORKSPACE_NOT_FOUND on any :slug route (access gone immediately)
→ if tx fails anywhere → full rollback, membership and projects unchanged (spec rule 10)
```

### 8.5 Leave workspace

```text
Member/Admin → POST /api/v1/workspaces/:slug/leave {confirm:true}
→ service asserts caller.role !== OWNER else 409 TRANSFER_REQUIRED
→ same tx shape as §8.4 (transfer + delete) but target is caller
→ 200 { transferredProjects } → client navigates to /select-workspace or /onboarding
```

### 8.6 Transfer ownership

```text
Owner → POST /api/v1/workspaces/:slug/transfer-ownership {targetMemberId}
→ resolveWorkspaceContext OWNER ✓ → service tx {
     assert caller is current Owner (re-read inside tx)
     assert target exists in same workspace, target !== caller, target.role ∈ {MEMBER, ADMIN}
     // swap — must be atomic with respect to workspace_single_owner partial index
     // correct pattern: single raw UPDATE ... CASE or deferrable index;
     // naive two sequential UPDATEs would transiently see two Owners per-statement
     UPDATE workspace_member SET role = CASE id WHEN :callerId THEN 'ADMIN' WHEN :targetId THEN 'OWNER' END
       WHERE id IN (:callerId, :targetId)
   } → 200 updated members
→ Owner is now Admin, target is now Owner — exactly one Owner still holds (index checked at commit)
→ if target left between page load and confirm → 409 TRANSFER_TARGET_INVALID, both retain original roles
```

### 8.7 Resend / revoke

```text
Owner → POST .../invitations/:invitationId/resend {confirm:true}
→ service asserts invitation.workspaceId === context.workspaceId && status===PENDING && expiresAt>now()
→ resend same token via Resend, bump updatedAt → 200 card

Owner → POST .../invitations/:invitationId/revoke {confirm:true}
→ UPDATE status=REVOKED where status=PENDING → 200 card (REVOKED)
→ subsequent GET /api/v1/invitations/:token preview → 200 with status=REVOKED
→ subsequent POST .../accept → 409 INVITATION_NOT_USABLE
```

### 8.8 Decline & expiry

```text
Invitee → POST /api/v1/invitations/:token/decline {} (verified session)
→ UPDATE status=DECLINED where status=PENDING → 200

Ignored invitation → expiresAt passes → no row mutation required
→ GET /api/v1/invitations/:token still 200 but service computes status=EXPIRED for display
→ POST .../accept → 409 INVITATION_EXPIRED
→ optional periodic job may flip PENDING→EXPIRED for list hygiene — correctness never depends on it
```

---

## 9. Module layout

### 9.1 API — `apps/api/src/features/members/`

```text
features/members/
├── routes.ts        # router: path defs → middleware chain → controller; Zod validated at entry
│                    # mounts both workspace-scoped and token-gated routers
├── schemas.ts       # route-local param/query coercion; shared request/response shapes live in packages/shared
├── controller.ts    # HTTP concerns only: parse req, call service, map result/errors to responses
├── service.ts       # business rules, permission matrix, state transitions, transactions
│                    # (invite batch tx, accept tx, transfer ownership swap, remove/leave + project transfer)
├── repository.ts    # Prisma access only; no business decisions leak here
└── errors.ts        # typed domain errors → global handler maps to codes in §7
```

Shared guards reused (not owned by this module):

```text
common/guards/
├── require-session.ts           # (F1) — session + emailVerified gate
├── workspace-context.ts         # (F2) — resolveWorkspaceContext(:slug)
└── require-workspace-role.ts    # (F2) — role check against resolved context
```

> Naming: the repo uses `src/features/…`; `Implementation Plan.md` §5 Step 4 sketches `modules/<feature>/`. Same concept — sync the plan's wording to `features/` rather than renaming code.

### 9.2 Shared — `packages/shared/src/members/`

Re-exports from `data-model.md` §4 — the canonical place:

- Enums: `workspaceRoleSchema` (widened), `invitationStatusSchema`, `INVITATION_TTL_DAYS`
- Request: `inviteMembersSchema`, `resendInvitationSchema`, `revokeInvitationSchema`, `changeMemberRoleSchema`, `removeMemberSchema`, `transferOwnershipSchema`, `confirmActionSchema`
- Response: `workspaceMemberCardSchema`, `invitationCardSchema`, `invitationPreviewSchema`

All Zod schemas are the single source of truth — API validates at the edge, web builds forms/mutations from the same types.

### 9.3 Web — `apps/web`

| Surface | Route | Reads/Writes |
|---|---|---|
| Members directory | `/w/:slug/members` | #1 list (all roles) |
| Member details drawer | `/w/:slug/members` (drawer over list) | #2 get, #3 change role, #4 remove, #6 transfer |
| Invite modal | `/w/:slug/members` → modal | #7 invite |
| Pending invitations list | `/w/:slug/members` (tab/section) | #8 list, #9 resend, #10 revoke |
| Invitation preview / accept / decline | `/invite/:token` | #11 preview, #12 accept, #13 decline |
| Leave workspace | `User Menu → Leave` / `/w/:slug/settings` | #5 leave |
| Transfer ownership dialog | `/w/:slug/members` or `/w/:slug/settings` | #6 transfer |
| Workspace switcher | `App shell` | Reads `workspaceMember.role` for display (data-model §4 card) |

Data access via TanStack Query hooks; mutations are pessimistic for remove/leave/transfer/accept (no optimistic role flip — permissions must be authoritative). Directory polls or refetches on window focus; notification of invite arrival is via email, not WebSocket (arch §11).

All surfaces ship with: loading, error, empty (no members besides owner; no pending invites), and permission-aware states (invite button hidden for `MEMBER`, resend/revoke hidden for `MEMBER`, role menu hidden for non-Owner).

---

## 10. Testing strategy

Three layers, each owned where it belongs (maps to plan §5 Step 6). All tooling provisioned by F1/F2 — no new dependencies.

### 10.1 API integration tests

Supertest against `app.ts` (or `createApp()`), real Postgres via Testcontainers, migrations applied by `vitest.global-setup.ts`. Seeded helpers: `createVerifiedUser`, `createWorkspaceAs(ownerUser)`, `addMember(workspace, user, role)`, `createInvitation(workspace, email, role, status)`.

| Case | Covered by |
|---|---|
| Happy paths ×13 endpoints | Supertest suite per route group (members, invitations-workspace, invitations-token) |
| Invalid input (bad email, empty batch, batch >20, bad role, missing confirm, bad memberId) | per-endpoint `400 VALIDATION_ERROR` |
| Unauthenticated access ×13 | `401 UNAUTHENTICATED` |
| Non-member workspace access (real slug, foreign user) | `404 WORKSPACE_NOT_FOUND` identical to unknown-slug — **assert byte-equal** (leak test) |
| Wrong role — invite as Admin by Admin | `403 FORBIDDEN_ROLE` |
| Wrong role — change role by Admin/Member | `403` |
| Wrong role — list invitations as Member | `403` |
| Owner cage — remove Owner | `409 CANNOT_REMOVE_OWNER` |
| Owner cage — Owner leave without transfer | `409 TRANSFER_REQUIRED` |
| Owner cage — demote Owner via role patch | `409 CANNOT_CHANGE_OWNER_ROLE` |
| Self-invite | `409 CANNOT_INVITE_SELF` |
| Already member | `409 ALREADY_MEMBER` |
| Pending exists (resend instead) | `409 PENDING_EXISTS` |
| Invite batch atomicity — one email already pending | none inserted, `409` with per-email details |
| Accept — happy, creates membership + flips to ACCEPTED | `201` + DB assertions |
| Accept — unverified user | `403 EMAIL_NOT_VERIFIED` |
| Accept — expired (`expiresAt` in past) | `409 INVITATION_EXPIRED` |
| Accept — revoked/declined/already accepted | `409 INVITATION_NOT_USABLE` |
| Accept — already a member | `409 ALREADY_MEMBER` |
| Accept — workspace archived | `409 WORKSPACE_ARCHIVED` |
| Accept — race: two concurrent accepts on same token | one `201`, one `409` |
| Decline — happy | `200` `DECLINED` |
| Decline — on expired/revoked | `409` |
| Resend — happy, bumps updatedAt, same token | DB assertion token unchanged |
| Revoke — happy | `200` `REVOKED` |
| Revoke — on already accepted | `409 INVITATION_NOT_USABLE` |
| Transfer — happy Owner→Admin swap, exactly one Owner remains | DB + index assertion |
| Transfer — target left concurrently | `409 TRANSFER_TARGET_INVALID`, both retain original roles |
| Transfer — concurrent transfers (two Owners racing — not reachable but test the swap) | one `200`, one `409` or `403` |
| Remove — happy + transferredProjects count | `200` + count; Checkpoint A expects `0`, Checkpoint B asserts real count |
| Remove — Admin removing Admin | `403 FORBIDDEN_ROLE` |
| Remove — full rollback on project-transfer failure (Checkpoint B hook) | assert membership still exists |
| Leave — happy as Member/Admin | `200` + transfer count |
| Archived workspace writes (#3, #4, #5, #6, #7, #9, #10, #12) | `409 WORKSPACE_ARCHIVED` |
| Archived reads (#1, #8, #11 preview) | `200` |
| Cross-workspace — invitationId from another workspace | `404 INVITATION_NOT_FOUND` scoped |

### 10.2 Component tests (web) — UI behavior with mocked API

**Setup:** Vitest + jsdom + Testing Library; MSW intercepts `/api/v1/workspaces/:slug/members*`, `/api/v1/workspaces/:slug/invitations*`, `/api/v1/invitations/:token*` with per-test handlers whose payloads are built from the real `packages/shared` schemas.

| Surface | Cases |
|---|---|
| Members directory | Renders `workspaceMemberCardSchema` rows (name, email, role, join date); role badge shows `Owner`/`Admin`/`Member`; empty state when only owner |
| Permission-aware directory | Invite button hidden for `MEMBER`; role menu hidden for non-Owner; remove button hidden for insufficient role per matrix |
| Invite modal | Submit issues `POST #7` with schema-valid body (assert via MSW spy); invalid email shows inline error without request; Admin role option hidden when caller is Admin; success shows pending row |
| Pending list | Renders invitation cards; resend bumps `updatedAt` display; revoke removes from pending; `PENDING_EXISTS` error offers resend action |
| Change role dialog | Owner selects Member→Admin; confirm sends `PATCH #3`; success updates badge; `403`/`409` map to friendly messages |
| Remove dialog | Shows `transferredProjects` count; confirm sends `POST #4`; success removes row; archived workspace hides remove |
| Leave dialog | Shows `transferredProjects` count; Owner sees "transfer ownership first" instead of leave; confirm sends `POST #5` |
| Transfer ownership dialog | Owner picks target; confirm sends `POST #6`; success updates both role badges; `TRANSFER_TARGET_INVALID` shows retry |
| Invitation preview page | Renders preview schema; unverified user sees "verify email" CTA; expired/revoked/accepted show appropriate empty state; verified user sees Accept/Decline buttons |
| Accept/decline | Accept sends `POST #12` and navigates to `/w/:slug`; decline sends `POST #13` and shows confirmation |
| Error envelope rendering | Every surface renders MSW-served `{error:{code,message}}` as friendly states, never raw dumps |
| Archived workspace wrapper | All mutating affordances disabled with "workspace archived" messaging |

Rules: components must never re-implement business rules (plan §1.2 — e.g., "Admin cannot remove Admin" is API-enforced; web just hides the button). Tests assert wire behavior and rendered state, not internals.

### 10.3 End-to-end journey — golden paths kept green

Playwright (`pnpm --filter @shipyard/web test:e2e`) against the locally composed stack (web + api + Postgres, DB reset/re-migrated between runs). In local dev the invitation email is captured by the test email adapter (F1's mode) instead of Resend — the test reads the token from the captured email or directly from the DB via the `test` helper route (`apps/api/src/features/test/routes.ts` in non-prod).

**Journey A — invite + accept as existing verified user (core)**

```text
1. Owner signs up + verifies + creates workspace (F2 flow)
2. Owner invites bob@example.com as Member → pending row appears
3. Bob signs up + verifies (separate browser context)
4. Bob opens /invite/:token → preview shows workspace + Member role
5. Bob accepts → member directory now shows Bob as Member
6. Owner changes Bob Member → Admin → badge flips to Admin
7. Owner transfers ownership to Bob → Owner becomes Admin, Bob becomes Owner
8. New Owner (Bob) removes former Owner — allowed (Owner can remove Admin)
```

**Journey B — admin cannot invite admin, cannot remove admin**

```text
1. Owner invites carol as Admin → Carol accepts
2. Carol (Admin) tries to invite as Admin → rejected (403) — retries as Member → 201
3. Carol tries to remove Owner → 409 CANNOT_REMOVE_OWNER
4. Carol removes a Member → 200
5. Carol tries to change a role → 403 FORBIDDEN_ROLE
```

**Negative E2E checks (cheap, high-value):**

- **Old link reuse:** accept an invitation, then replay `POST .../accept` on same token → 409, no duplicate membership.
- **Owner cage:** Owner tries to leave without transfer → 409 TRANSFER_REQUIRED; after transfer, former Owner can leave as Admin.
- **Archived freeze:** archive workspace, then try to invite / change role / remove / accept a pending invite → 409 WORKSPACE_ARCHIVED; directory still readable.
- **Cross-workspace leak:** second workspace's invitationId used under first workspace's slug → 404 INVITATION_NOT_FOUND scoped.
- **Unverified accept:** create invitation, sign up but do not verify, try to accept → 403 EMAIL_NOT_VERIFIED.

Scope discipline: journeys A–B plus negatives are the mandatory E2E suite for F3 Checkpoint A; exhaustive cases stay in layers 10.1–10.2. Integration journey for Checkpoint B (remove/leave with real project transfer) lands after F4.

---

## 11. Cross-cutting concerns

| Concern | Approach |
|---|---|
| **Rate limiting** | Per-route: invite (5/min per workspace), resend (3/min per invitation), accept (10/min per IP). Memory for MVP (like Auth). Global limiter already on `/api/v1`. Detailed wiring finalized at F12. |
| **Email** | Resend adapter — `sendInvitationEmail(to, workspaceName, role, tokenLink)` called after commit (non-transactional; failure does not roll back the row). Local dev: test adapter captures token to memory/DB so E2E can read it without email. |
| **Audit** | No audit-log UI in MVP (spec §6). `invitation.createdById` + `createdAt`/`updatedAt`/`status` are the audit trail; future UI can read them from the same table. |
| **Pagination** | Not needed — member count small (spec §7 Q4 none, open). Invitation list small. If needed later, add cursor pagination reusing F1's pattern. |
| **Search** | Member directory search is client-side filter on the fetched list for MVP (small N); server-side search lands with F10 Search (Postgres full-text on `user.name`/`email`). |

---

## 12. References

- Shipyard: `features/members/spec.md`, `features/members/data-model.md`, `features/workspace/spec.md`, `features/workspace/data-model.md`, `features/workspace/api-design.md` (guard chain, archived matrix, envelope), `features/auth/data-model.md` (emailVerified), `features/auth/api-design.md` (Better Auth session, Resend pattern), `00-architecture.md` §5–§8, `ADR-001`–`ADR-003`, `Implementation Plan.md` F3
- Express middleware patterns & error handling follow existing `apps/api/src/common` utilities built in F1/F2
- Prisma: partial unique index, referential actions — `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- PostgreSQL partial index — `https://www.postgresql.org/docs/current/indexes-partial.html`
- Better Auth user model — `https://www.better-auth.com/docs/concepts/database`

---

*Next artifact: implementation itself (plan §5 Steps 3–7) — Prisma migration → module code (routes/controller/service/repository + shared schemas) → web slice (directory, invite, drawer, dialogs, `/invite/:token`) → tests → `pnpm check`. No further design doc needed; data-model + api-design cover F3's technical design. Checkpoint A ships first; Checkpoint B integration (project transfer) lands after F4.*
