# Issues — API Design

**Module:** `apps/api/src/modules/issues`
**Status:** Draft v0.1 — 2026-08-12
**Base URL:** `/api/v1` (proxied by Next.js; API internal-only)

---

## 1. Conventions

- **Resource:** `/api/v1/workspaces/{workspaceId}/issues` (+ `/labels`).
- **Guards:** `requireSession → requireWorkspaceMember`; delete adds `requireRole(OWNER | ADMIN)`.
- **View preference:** stored per user per workspace (settings module) — the API never guesses which view to render.
- **Pagination:** cursor-based (`cursor`, `limit` default 50 / max 100) on list endpoints.
- **Errors:** global envelope; codes in §4. Statuses: `200` · `201` · `204` · `400` · `401` · `403` · `404` · `409` · `429`.

---

## 2. Endpoint Map

| Method | Path | Domain op | Guard |
|---|---|---|---|
| POST | `/workspaces/{wsId}/issues` | createIssue | member |
| GET | `/workspaces/{wsId}/issues` | listIssues | member |
| GET | `/workspaces/{wsId}/issues/{issueId}` | getIssue (+ history) | member |
| PATCH | `/workspaces/{wsId}/issues/{issueId}` | updateIssue / changeStatus / toggleBlocked | member |
| POST | `/workspaces/{wsId}/issues/{issueId}/archive` | archiveIssue | member |
| POST | `/workspaces/{wsId}/issues/{issueId}/restore` | restoreIssue | member |
| DELETE | `/workspaces/{wsId}/issues/{issueId}` | deleteIssue | **Owner / Admin** |
| GET | `/workspaces/{wsId}/labels` | list labels | member |
| POST | `/workspaces/{wsId}/labels` | create label | member |

---

## 3. Endpoint Details

### 3.1 Create — `POST /workspaces/{wsId}/issues`

**Body (Zod):**
```ts
{
  title: string(1..200),            // required
  description?: string,
  priority?: "NO_PRIORITY"|"URGENT"|"HIGH"|"MEDIUM"|"LOW",  // default NO_PRIORITY
  projectId?: string,               // must exist in this workspace
  cycleId?: string,                 // must exist in this workspace
  assigneeId?: string,              // must be a member of this workspace
  labelIds?: string[],              // must exist in this workspace
  dueDate?: string,                 // ISO date (YYYY-MM-DD)
}
```

**Responses:** `201` `{ issue }` (identifier e.g. `SHIP-024`, status `BACKLOG`) · `400 ISSUE_INVALID_INPUT` · `403 ISSUE_FORBIDDEN` · `429 ISSUE_RATE_LIMITED`.

### 3.2 List — `GET /workspaces/{wsId}/issues`

**Query params:**
| Param | Values |
|---|---|
| `view` | `all` (default) · `mine` · `backlog` · `blocked` · `archived` |
| `status` | `BACKLOG`·`TODO`·`IN_PROGRESS`·`DONE` (multi) |
| `priority` | enum (multi) |
| `assigneeId` · `projectId` · `cycleId` | single ids |
| `labelIds` | multi |
| `dueDateFrom` · `dueDateTo` | date range |
| `blocked` | `true`/`false` |
| `q` | full-text search (title + description) |
| `sort` | `createdAt` (default) · `updatedAt` · `dueDate` · `priority` · `title` |
| `order` | `asc` · `desc` |
| `cursor` · `limit` | pagination |

**Responses:** `200` `{ issues: IssueCard[], nextCursor: string|null }` — `IssueCard` = identifier, title, priority, assignee, project, cycle, dueDate, isBlocked, status, labels.

### 3.3 Get — `GET /workspaces/{wsId}/issues/{issueId}`

**Responses:** `200` `{ issue: IssueDetail }` — full record + `activity: IssueActivity[]` (newest first) + `comments` summary pointer · `404 ISSUE_NOT_FOUND` (or not a member — identical response).

### 3.4 Update — `PATCH /workspaces/{wsId}/issues/{issueId}`

**Body:** any subset of create fields + `status` + `isBlocked` + `blockedReason`. At least one field.

**Business rules server-side:**
- `status: DONE` + blocked → auto-clear flag/reason (activity records both)
- `isBlocked: true` on DONE → `400 ISSUE_CANNOT_BLOCK`
- Archived issue → `409 ISSUE_ARCHIVED` (any write)
- Assignee change → notification side-effect (same transaction)

**Responses:** `200` `{ issue }` · `400` · `404` · `409` · `429`.

### 3.5 Archive / Restore

`POST /workspaces/{wsId}/issues/{issueId}/archive` → `200 { issue }` · `409` if already archived.
`POST /workspaces/{wsId}/issues/{issueId}/restore` → `200 { issue }` · `409` if active.

### 3.6 Delete — `DELETE /workspaces/{wsId}/issues/{issueId}`

**Guards:** member + `requireRole(OWNER | ADMIN)`.

**Responses:** `204` · `403 ISSUE_FORBIDDEN` (member) · `404`.

### 3.7 Labels

`GET /workspaces/{wsId}/labels` → `200 { labels: [{ id, name, color }] }`.
`POST /workspaces/{wsId}/labels` body `{ name, color? }` → `201 { label }` · `409 LABEL_ALREADY_EXISTS` (normalized name conflict).

---

## 4. Error Codes (issues domain)

| Code | Status | Meaning |
|---|---|---|
| `ISSUE_INVALID_INPUT` | 400 | Zod validation / cross-workspace reference / block-on-DONE |
| `ISSUE_CANNOT_BLOCK` | 400 | Blocking a DONE issue |
| `ISSUE_UNAUTHORIZED` | 401 | No valid session |
| `ISSUE_FORBIDDEN` | 403 | Not a member / member attempting delete |
| `ISSUE_NOT_FOUND` | 404 | Unknown id or not in workspace (identical response) |
| `ISSUE_ARCHIVED` | 409 | Write on an archived issue |
| `ISSUE_CONFLICT` | 409 | Concurrent change (drag refresh path) / state conflict |
| `LABEL_ALREADY_EXISTS` | 409 | Normalized label name conflict |
| `ISSUE_RATE_LIMITED` | 429 | Rate limit hit |

---

## 5. Rate Limiting

| Endpoint | Limit |
|---|---|
| `POST /issues` · `PATCH /issues/{id}` · archive/restore/delete | 60/min per user |
| `GET /issues` · `GET /issues/{id}` | 120/min per user |
| `POST /labels` | 30/min per user |

---

## 6. Web Integration (consumers)

| Web surface | Endpoint(s) |
|---|---|
| Issues page — List view | `GET /issues?view=all` (+ filters/sort) |
| Issues page — Kanban | Same query, grouped by status client-side from `view=all` |
| My Issues (list/kanban) | `GET /issues?view=mine` |
| Backlog / Blocked / Archived views | `GET /issues?view=backlog|blocked|archived` |
| Create Issue modal (issues page, project, cycle, dashboard, global create) | `POST /issues` |
| Issue Details page | `GET /issues/{id}` · `PATCH /issues/{id}` |
| Status dropdown | `PATCH /issues/{id}` `{ status }` |
| Kanban drag | Same endpoint (optimistic + refresh-on-409) |
| Blocked toggle (details + list row) | `PATCH /issues/{id}` `{ isBlocked, blockedReason? }` |
| Archive/Restore (details + archived view) | `archive` · `restore` |
| Delete (details, Owner/Admin only) | `DELETE /issues/{id}` |
| Label management (filter bar / issue modal) | `GET/POST /labels` |
| Global search (search module) | Reuses the issue repository via internal service, not this endpoint |

All calls go **through the Next proxy** (ADR-003).

---

## 7. OpenAPI & Shared Contracts

- Zod schemas in `packages/shared`: `CreateIssueInput`, `UpdateIssueInput`, `IssueListQuery`, `IssueCard`, `IssueDetail`, `IssueActivity`, `LabelInput`, `Label` — shared with web + OpenAPI generation.
- `IssueStatus` / `IssuePriority` enums defined in `packages/shared` — imported by Prisma-facing code and the web (one source of truth).
- Query-string schemas are strict (unknown params rejected) — keeps the API contract tight.

---

## 8. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Priority color mapping | Semantic colors exist (design system); map URGENT→danger, HIGH→warning, MEDIUM→info? Decide in UI implementation |
| 2 | Activity granularity | Currently one row per change; if the dashboard feed needs "created/completed" only, the `action` enum covers it without schema change |
| 3 | Bulk operations | No bulk edit/archive in MVP (PRD silent) — revisit post-MVP |

