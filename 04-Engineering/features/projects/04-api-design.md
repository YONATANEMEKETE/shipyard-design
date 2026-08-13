# Projects — API Design

**Module:** `apps/api/src/modules/projects`
**Status:** Draft v0.1 — 2026-08-12
**Base URL:** `/api/v1` (proxied by Next.js; API internal-only)

---

## 1. Conventions

- **Resource:** `/api/v1/workspaces/{workspaceId}/projects`.
- **Guards:** `requireSession → requireWorkspaceMember`; create/transfer/archive/restore/delete add `requireRole(OWNER | ADMIN)`; edits are member-level (PRD matrix).
- **Project issues** are listed via the **issues endpoint** (`GET /issues?projectId=…`) — no duplicated nested list here (DRY contract).
- **Errors:** global envelope; codes in §4. Statuses: `200` · `201` · `204` · `400` · `401` · `403` · `404` · `409` · `429`.

---

## 2. Endpoint Map

| Method | Path | Domain op | Guard |
|---|---|---|---|
| POST | `/workspaces/{wsId}/projects` | createProject | **Owner / Admin** |
| GET | `/workspaces/{wsId}/projects` | listProjects | member |
| GET | `/workspaces/{wsId}/projects/{projectId}` | getProject (+ activity) | member |
| PATCH | `/workspaces/{wsId}/projects/{projectId}` | updateProject (incl. free status switch) | member |
| POST | `/workspaces/{wsId}/projects/{projectId}/transfer-ownership` | transferOwnership | **Owner / Admin** |
| POST | `/workspaces/{wsId}/projects/{projectId}/archive` | archiveProject | **Owner / Admin** |
| POST | `/workspaces/{wsId}/projects/{projectId}/restore` | restoreProject | **Owner / Admin** |
| DELETE | `/workspaces/{wsId}/projects/{projectId}` | deleteProject | **Owner / Admin** |

---

## 3. Endpoint Details

### 3.1 Create — `POST /workspaces/{wsId}/projects`

**Body (Zod):** `{ name: string(1..64), description?: string, startDate?: date, targetDate?: date }`

**Responses:**
- `201` — `{ project: { id, name, status: "PLANNED", ownerId, startDate, targetDate, progress, createdAt } }` (creator = owner)
- `400` — `PROJECT_INVALID_INPUT` · `409` — `PROJECT_NAME_CONFLICT` (normalized; archived projects reserve names) · `403` — `PROJECT_FORBIDDEN` · `429`

### 3.2 List — `GET /workspaces/{wsId}/projects`

**Query params:** `status` (multi) · `ownerId` · `startDateFrom/To` · `targetDateFrom/To` · `sort` (`createdAt` default · `updatedAt` · `startDate` · `targetDate` · `name`) · `order`.

**Responses:** `200` `{ projects: ProjectCard[] }` — `ProjectCard` = name, owner, status, progress (`{ total, completed, percent }`), targetDate. **Archived excluded** (separate list, PRD). No pagination in MVP (decided).

### 3.3 Get — `GET /workspaces/{wsId}/projects/{projectId}`

**Responses:** `200` `{ project: ProjectDetail }` — full record + progress + `activity: ProjectActivity[]` + `issueSummary: { total, completed, blocked }` + `cycles: [{ id, name }]` (**derived** from the project's issues — never stored on the project) · `404 PROJECT_NOT_FOUND` (or not a member — identical response).

### 3.4 Update — `PATCH /workspaces/{wsId}/projects/{projectId}`

**Body:** any subset of `{ name, description?, startDate?, targetDate?, status? }`.

**Business rules server-side:**
- Archived → `409 PROJECT_ARCHIVED` (any write)
- `status` ∈ {PLANNED, ACTIVE, COMPLETED} only — ARCHIVED → `400`
- Name change → normalized conflict check → `409 PROJECT_NAME_CONFLICT`

**Responses:** `200` `{ project }` · `400` · `404` · `409` · `429`.

### 3.5 Transfer ownership — `POST /workspaces/{wsId}/projects/{projectId}/transfer-ownership`

**Body:** `{ userId: string }` — any current workspace member (Owner/Admin/Member; not self-needed check — same target is a no-op).

**Responses:** `200` `{ project }` (ownerId updated; recipient's workspace role unchanged) · `403` (non-Owner/Admin) · `404` (target no longer a member — re-verified in-txn) · `409` (archived).

### 3.6 Archive / Restore

`POST …/archive` → `200 { project }` (status column untouched — restore target) · `409` if already archived.
`POST …/restore` → `200 { project }` (back to stored status) · `409` if active.

### 3.7 Delete — `DELETE /workspaces/{wsId}/projects/{projectId}`

**Responses:** `204` (+ `X-Unassigned-Issues: N` header) — issues survive with projectId cleared atomically · `403` (member) · `404`.

---

## 4. Error Codes (projects domain)

| Code | Status | Meaning |
|---|---|---|
| `PROJECT_INVALID_INPUT` | 400 | Zod validation / ARCHIVED sent as status |
| `PROJECT_UNAUTHORIZED` | 401 | No valid session |
| `PROJECT_FORBIDDEN` | 403 | Member attempting Owner/Admin-only action |
| `PROJECT_NOT_FOUND` | 404 | Unknown id, not in workspace, or transfer target gone (identical response) |
| `PROJECT_ARCHIVED` | 409 | Write on an archived project |
| `PROJECT_NAME_CONFLICT` | 409 | Normalized name conflict (create/rename; archived reserve names) |
| `PROJECT_RATE_LIMITED` | 429 | Rate limit hit |

---

## 5. Rate Limiting

| Endpoint | Limit |
|---|---|
| `POST /projects` · `PATCH /projects/{id}` · transfer/archive/restore/delete | 60/min per user |
| `GET /projects` · `GET /projects/{id}` | 120/min per user |

---

## 6. Web Integration (consumers)

| Web surface | Endpoint(s) |
|---|---|
| Projects page — List view (Active/Planned/Completed sections) | `GET /projects` |
| Projects page — Kanban (3 columns) | Same query, grouped server-side |
| Archived Projects list | `GET /projects?status=…` is excluded — archived via the dedicated Archived section (reuses list endpoint with archived filter — see open question 1) |
| Create Project modal (projects page, global create — Owner/Admin) | `POST /projects` |
| Project Details page | `GET /projects/{id}` · `PATCH /projects/{id}` |
| Project issues tab | `GET /issues?projectId={id}` (issues endpoint — no duplicate) |
| Change Project Owner (Owner/Admin) | `transfer-ownership` |
| Status dropdown / kanban drag | `PATCH /projects/{id}` `{ status }` |
| Archive / Restore (confirmed dialogs) | `archive` · `restore` |
| Delete (confirmed dialog with issue count) | `DELETE /projects/{id}` |

All calls go **through the Next proxy** (ADR-003).

---

## 7. OpenAPI & Shared Contracts

- Zod schemas in `packages/shared`: `CreateProjectInput`, `UpdateProjectInput`, `TransferProjectOwnershipInput`, `ProjectCard`, `ProjectDetail`, `ProjectActivity` — shared with web + OpenAPI generation.
- `ProjectStatus` enum in `packages/shared`; `ProjectId` cuid type shared as a route param.

---

## 8. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Archived projects list endpoint | Propose `GET /projects?archived=true` (single list endpoint, archived filter) — confirm before implementation |
| 2 | Delete confirmation count | Returned via `X-Unassigned-Issues` header — confirm web reads it for the success toast |
| 3 | Project search | `q` param on project list is out of MVP scope (global search covers it via search module) — confirm |

