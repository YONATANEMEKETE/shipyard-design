# Workspace — API Design

**Module:** `apps/api/src/modules/workspace`
**Status:** Draft v0.1 — 2026-08-12
**Base URL:** `/api/v1` (proxied by Next.js; API internal-only)

---

## 1. Conventions

- **Resource:** `/api/v1/workspaces` — the workspace resource and its lifecycle actions.
- **Auth:** every endpoint requires a session (`requireSession`); lifecycle actions additionally require `requireRole(OWNER)`; access requires `requireWorkspaceMember`.
- **Body:** JSON; all schemas in `packages/shared` (Zod).
- **Success shape:** the resource or `{}`; errors use the global envelope:
  ```json
  { "error": { "code": "WORKSPACE_STATE_CONFLICT", "message": "...", "details": {} } }
  ```
- **Status codes:** `200` · `201` (create) · `204` (delete) · `400` validation · `401` unauthenticated · `403` forbidden · `404` not found · `409` state conflict · `429` rate limited.
- **Names are never identifiers:** request bodies carry display names; routes carry the workspace **id**. No endpoint accepts a name as a lookup key.

---

## 2. Endpoint Map

| Method | Path | Domain op | Role | Body / Query |
|---|---|---|---|---|
| POST | `/workspaces` | createWorkspace | verified user | `{ name, icon? }` |
| GET | `/workspaces` | getWorkspaces | member | — |
| GET | `/workspaces/{id}` | getWorkspace | member | — |
| GET | `/workspaces/archived` | getArchivedWorkspaces | **OWNER** | — |
| PATCH | `/workspaces/{id}` | updateWorkspace | **OWNER** | `{ name?, icon? }` |
| POST | `/workspaces/{id}/archive` | archiveWorkspace | **OWNER** | — |
| POST | `/workspaces/{id}/restore` | restoreWorkspace | **OWNER** | — |
| POST | `/workspaces/{id}/delete` | deleteWorkspace | **OWNER** | `{ confirmName }` |

*`GET /workspaces/{id}` exists for deep-link and notification navigation — the client needs one workspace's context (name, icon, role, status) to render the shell without fetching the full list.*

### 3.2b Single — `GET /workspaces/{id}`

**Guards:** member (404 for non-members — same response as unknown id, no existence leak).

**Responses:**
- `200` — `{ workspace: { id, name, icon, status, role, isOwner, archivedAt, createdAt } }` — includes **the caller's role** and ownership flag so the shell renders permission-aware UI immediately (archived state included for the Owner's summary page)
- `401` — `WORKSPACE_UNAUTHORIZED` · `404` — `WORKSPACE_NOT_FOUND`

**Consumers:** deep links (notification navigation, bookmarks, refresh), app-shell context on first paint, archived workspace summary page.

---

## 3. Endpoint Details

### 3.1 Create — `POST /workspaces`

**Body:** `{ name: string, icon?: string }`
**Validation (Zod):** `name` trimmed, 1–64 chars · `icon` optional, must be a signed R2 URL (server-issued earlier via the upload flow) or null.

**Responses:**
- `201` — `{ workspace: { id, name, icon, status: "ACTIVE", ownerId, createdAt } }` → web redirects to `/workspaces/{id}/dashboard`
- `400` — `WORKSPACE_INVALID_INPUT` (validation details)
- `429` — `WORKSPACE_RATE_LIMITED`

**Behavior:** single transaction — workspace row + Owner membership row; both or neither. Creator becomes Owner. Duplicate names accepted silently (no uniqueness check).

### 3.2 List — `GET /workspaces`

**Responses:**
- `200` — `{ workspaces: [{ id, name, icon, role, status, isOwner, updatedAt }] }` — all memberships of the session user, including owned archived workspaces (marked `status: "ARCHIVED"`, `isOwner: true`)

**Consumers:** login resolver (0 → onboarding, 1 → auto-enter, n → selection), workspace switcher modal, archived section (Owner). Order: active first, then by `updatedAt` desc.

### 3.3 Archived list — `GET /workspaces/archived`

**Responses:**
- `200` — `{ workspaces: [{ id, name, icon, archivedAt }] }` — **Owner-only** (403 for others; members see no archived section at all).

### 3.4 Update — `PATCH /workspaces/{id}`

**Body:** `{ name?, icon? }` — at least one field; both optional but not both empty.
**Guards:** member + **OWNER** · workspace must be **ACTIVE** (409 if archived).

**Responses:**
- `200` — `{ workspace }` updated
- `400` — `WORKSPACE_INVALID_INPUT`
- `403` — `WORKSPACE_FORBIDDEN` (not Owner)
- `404` — `WORKSPACE_NOT_FOUND` (or not a member — same response, no existence leak)
- `409` — `WORKSPACE_STATE_CONFLICT` (archived)

### 3.5 Archive — `POST /workspaces/{id}/archive`

**Guards:** member + **OWNER** · must be ACTIVE.

**Responses:** `200` `{ workspace }` (status `ARCHIVED`) · `403` · `404` · `409` if already archived.

### 3.6 Restore — `POST /workspaces/{id}/restore`

**Guards:** member + **OWNER** · must be ARCHIVED.

**Responses:** `200` `{ workspace }` (status `ACTIVE`, `archivedAt: null`) · `403` · `404` · `409` if already active.

### 3.7 Delete — `POST /workspaces/{id}/delete`

**Body:** `{ confirmName: string }`
**Guards (four gates, in order):** member + **OWNER** → must be ARCHIVED → **server-side exact-name match** against `workspace.name`.

**Responses:**
- `204` — deleted (cascade removes all scoped data atomically)
- `400` — `WORKSPACE_CONFIRM_MISMATCH` (name did not match; dialog stays open)
- `403` — `WORKSPACE_FORBIDDEN` (not Owner)
- `404` — `WORKSPACE_NOT_FOUND`
- `409` — `WORKSPACE_STATE_CONFLICT` (not archived — active workspaces cannot be deleted directly)

---

## 4. Error Codes (workspace domain)

| Code | Status | Meaning |
|---|---|---|
| `WORKSPACE_INVALID_INPUT` | 400 | Zod validation failed |
| `WORKSPACE_CONFIRM_MISMATCH` | 400 | Delete confirmation name ≠ workspace name |
| `WORKSPACE_UNAUTHORIZED` | 401 | No valid session |
| `WORKSPACE_FORBIDDEN` | 403 | Not a member / not Owner for the action |
| `WORKSPACE_NOT_FOUND` | 404 | Id unknown or not a member (identical response) |
| `WORKSPACE_STATE_CONFLICT` | 409 | Action invalid for current lifecycle state (archive active, delete non-archived, restore non-archived) |
| `WORKSPACE_RATE_LIMITED` | 429 | Rate limit hit (`Retry-After` header) |

---

## 5. Rate Limiting

| Endpoint | Limit |
|---|---|
| `POST /workspaces` | 10/min per user |
| `PATCH /workspaces/{id}` | 20/min per user |
| `archive` / `restore` / `delete` | 10/min per user each |
| `GET /workspaces` (+ archived) | 60/min per user |

---

## 6. Web Integration (consumers)

| Web surface | Endpoint(s) |
|---|---|
| Onboarding screen (`Auth / Signup` → first login) | `POST /workspaces` · `GET /workspaces` (resolver) |
| Workspace selection screen | `GET /workspaces` |
| Workspace switcher modal | `GET /workspaces` |
| Archived Workspaces section (switcher, Owner) | `GET /workspaces/archived` |
| Workspace Settings → Details | `PATCH /workspaces/{id}` |
| Workspace Settings → Danger Zone | `archive` · `restore` · `delete` (confirm dialogs) |
| All workspace pages (context) | Resolved via URL id + server-side membership check on every scoped request |

All calls go **through the Next proxy** — the browser never hits the API origin directly (ADR-003).

---

## 7. OpenAPI & Shared Contracts

- Zod schemas (`CreateWorkspace`, `UpdateWorkspace`, `WorkspaceResponse`, `WorkspaceListResponse`, `DeleteWorkspaceConfirm`) live in `packages/shared` → auto-generated OpenAPI types.
- The workspace id type (`WorkspaceId = z.string().cuid()`) is shared with every other module's schemas — one source of truth for route params.

---

## 8. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Icon upload endpoint | Lives in `settings`/shared upload flow (R2 presigned vs proxied — proxied per ADR-004); workspace API only accepts the resulting URL |
| 2 | List pagination | Workspaces per user are few — no pagination in MVP (note for future) |
| 3 | Delete: hard vs scheduled purge | Hard delete immediately (PRD: irreversible); no soft-delete tombstone for workspaces |

