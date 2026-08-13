# Workspace — Request Lifecycle

**Module:** `apps/api/src/modules/workspace`
**Status:** Draft v0.1 — 2026-08-12
**Relies on:** ADR-003 (Next.js proxy) · `02-data-model.md` · `04-api-design.md`

---

## 1. Overview

Workspace traffic is **JSON API + page loads** (no redirect flows to external parties). Two flows deserve special attention:

1. **Create** — a two-row transaction (workspace + owner membership) that must never half-apply.
2. **Permanent delete** — the most destructive operation in the system: exact-name gate + one atomic cascade DELETE.

**Key architectural decision — the active workspace is NOT server state.** The "current workspace" lives in the URL (`/workspaces/{id}/...`), not in the session. Every request independently resolves + validates the workspace id in its path. No server-side "active workspace" to sync, no stale context after switching — the URL *is* the context.

---

## 2. The global pipeline + workspace layer

```
Browser
  │  https://shipyard.yonatanem.com/workspaces/{id}/issues
  ▼
Caddy (:443) ──▶ Next.js (:3000) ──rewrite /api/*──▶ Express API (:4000)
                                                      │
   ┌──────────────────────────────────────────────────┘
   │  shared kernel middleware (every request):
   │    request-id → Pino log → security headers → rate limit → body parsing
   │
   ▼
   workspace layer:
   1. requireSession                 → 401 → web redirects to /login
   2. workspaceId from route params  → missing → 404 (not 400: don't leak structure)
   3. requireWorkspaceMember         → membership lookup (indexed)
        ├── not a member            → 403 WORKSPACE_FORBIDDEN → web redirects
        │                             to the user's own workspace (or onboarding)
        ├── workspace ARCHIVED + non-owner → 403 (see §9)
        └── member                  → attach req.workspace (id, status, ownerId, role)
   4. role guard where required      → requireRole(OWNER) for lifecycle actions
   5. route handler → service → repository → Prisma → Neon
```

**Trust boundary:** the workspace id comes from the URL — but it is **never trusted**; it is only a lookup key. Access is decided by the membership check against the session's `userId` (domain rule 6: membership is the only door).

## 3. Flow — Create workspace (JSON, from onboarding or switcher)

```
1. POST /api/v1/workspaces
   body: { name, icon? }                        [Zod: name 1-64 trimmed, icon URL or null]
2. requireSession                               [verified user — auth gate]
3. TRANSACTION (single):
   a. INSERT Workspace (status = ACTIVE, ownerId = user.id)
   b. INSERT Membership (workspaceId, userId, role = OWNER)
   — both rows or neither; the Owner invariant is born inside this transaction
4. 201 → web redirects to the new workspace dashboard
   (/workspaces/{id}/dashboard)
```

**On failure:** the whole transaction rolls back — no orphan workspace without an owner, no membership without a workspace. Retry is safe (idempotent outcome).

## 4. Flow — Enter workspace / switcher (resolve)

```
AFTER LOGIN (auth module hands off):
1. GET /api/v1/workspaces   (all memberships for session user)
2. Resolver logic (server-side, deterministic):
   ├── 0 workspaces        → web shows onboarding
   ├── 1 workspace         → redirect to its dashboard (auto-enter)
   └── n workspaces        → web shows selection screen
       (each entry: name + icon + role/ownership badge — duplicate
        names are distinct entries; selection uses the workspace ID)

DURING A SESSION (switcher modal):
3. GET /api/v1/workspaces   → same list, rendered in the modal
4. User picks an entry → client navigates to /workspaces/{id}/dashboard
   — no server round-trip for "switching": the new URL is the new context
```

## 5. Flow — Every workspace-scoped request (the hot path)

This runs on **every** request carrying a workspace id — issues, projects, cycles, members, settings, search:

```
route with { workspaceId }
  → requireSession (session → userId)
  → requireWorkspaceMember:
      SELECT membership WHERE workspaceId = $1 AND userId = $2   [unique index, O(1)]
      ├── row missing      → 403 WORKSPACE_FORBIDDEN
      ├── workspace archived + user ≠ owner → 403 (read-only boundary, §9)
      └── row found        → attach req.workspace { id, status, ownerId, role }
  → role guard (only where the PRD matrix requires): requireRole(OWNER | ADMIN)
  → module service → repository — every query additionally filters by req.workspace.id
```

**Why this shape:** membership is checked per request, not cached client-side — a removed member loses access on their very next request (PRD: "removed members immediately lose access"). Role changes apply immediately for the same reason.

## 6. Flow — Update workspace (rename / icon)

```
1. PATCH /api/v1/workspaces/{id}
   body: { name?, icon? }                       [Zod; name 1-64 trimmed]
2. requireSession → requireWorkspaceMember → requireRole(OWNER)
3. UPDATE Workspace (name / icon) — no uniqueness check (duplicates legal)
4. 200 → updated workspace; every screen reflects it (id unchanged — no
   references break, deep links keep working)
```

## 7. Flow — Archive / Restore

```
ARCHIVE:
1. POST /api/v1/workspaces/{id}/archive          (confirm dialog client-side)
2. requireSession → requireWorkspaceMember → requireRole(OWNER)
3. workspace must be ACTIVE                      [else 409 WORKSPACE_STATE_CONFLICT]
4. UPDATE status = ARCHIVED, archivedAt = now
5. 200 → web moves the workspace to the Archived section; it is now read-only
   (enforced at the data layer: write routes reject archived workspaces)

RESTORE:
1. POST /api/v1/workspaces/{id}/restore          (confirm dialog client-side)
2. same guards (OWNER)
3. workspace must be ARCHIVED                    [else 409]
4. UPDATE status = ACTIVE, archivedAt = null
5. 200 → web returns it to the active list; data and history untouched
```

## 8. Flow — Permanent delete (danger zone)

```
1. POST /api/v1/workspaces/{id}/delete
   body: { confirmName }                        [Zod: non-empty string]
2. requireSession → requireWorkspaceMember → requireRole(OWNER)
3. workspace must be ARCHIVED                   [else 409 WORKSPACE_STATE_CONFLICT
                                                  — active workspaces can't delete]
4. SERVER-SIDE name check: confirmName must EXACTLY equal workspace.name
   [else 400 WORKSPACE_CONFIRM_MISMATCH — the client dialog types the name,
    but the server is the one that enforces it; no client-trusted flag]
5. Single DELETE FROM Workspace WHERE id = $1
   → Postgres cascade removes memberships, invitations, projects, cycles,
     issues, comments, notifications (atomic by ACID, §5 of the data model)
6. 204 → web returns to onboarding / another workspace
```

**Safety layering:** Owner-only (role guard) → archived-only (state guard) → exact-name (input guard) → atomic cascade (DB guard). Four independent gates before a single DELETE.

## 9. Flow — Archived workspace access

```
DIRECT URL to an archived workspace (e.g. /workspaces/{id}/issues):
  → requireWorkspaceMember:
      ├── requester IS the Owner → read-only "Archived Workspace Summary"
      │     page: restore + delete actions available
      └── requester is a member  → 403 → web redirects to the user's
            active workspace with a notice ("workspace is archived")

SWITCHER (Owner only): "Archived Workspaces" section lists owned archived
  workspaces → opening one lands on the same summary page.
```

Write routes (create/update/archive/delete of any scoped resource) reject
archived workspaces at the data layer — read-only is enforced server-side,
not just in the UI.

## 10. Edge Cases & Failure Handling

| Case | Behavior |
|---|---|
| Not a member hits a workspace URL | 403 `WORKSPACE_FORBIDDEN` → redirect to own workspace / onboarding |
| Removed member keeps an old URL | 403 on the very next request (per-request membership check) |
| Archived workspace via URL (member) | 403 + redirect with notice |
| Owner deletes an active workspace | 409 `WORKSPACE_STATE_CONFLICT` — must archive first |
| Wrong confirmation name | 400 `WORKSPACE_CONFIRM_MISMATCH` — dialog stays open |
| Delete fails mid-cascade | Single transaction — everything rolls back; workspace remains Archived |
| Duplicate names in switcher | Distinct entries, selection by id — impossible to pick the wrong one |
| Rename while another user has the old name in a URL | Harmless: id-based routing, name is display-only |
| Ownership transferred while user is mid-session | Their role badge updates on next request; new Owner sees admin surfaces immediately |
| Rate limit | Same shared middleware as auth (per-IP); lifecycle actions 10/min |

## 11. Dev vs Prod Differences

| Concern | Local dev | Production |
|---|---|---|
| Origin / base URL | `http://localhost:3000` | `https://shipyard.yonatanem.com` |
| Icon uploads | R2 dev bucket or local stub | R2 prod bucket (private, signed URLs) |
| Rate limits | Same (catches abuse early) | Same |
| Onboarding redirects | Same logic | Same logic (no external URLs involved) |


