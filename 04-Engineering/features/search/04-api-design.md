# Search — API Design

**Module:** `apps/api/src/modules/search`
**Status:** Draft v0.1 — 2026-08-12
**Base URL:** `/api/v1` (proxied by Next.js; API internal-only)

---

## 1. Conventions

- **Resource:** `/api/v1/workspaces/{workspaceId}/search` — one read-only endpoint for all search shapes (full + suggestions).
- **Guards:** `requireSession → requireWorkspaceMember`.
- **Errors:** global envelope; codes in §4. Statuses: `200` · `400` · `401` · `403` · `429`.

---

## 2. Endpoint Map

| Method | Path | Domain op | Guard |
|---|---|---|---|
| GET | `/workspaces/{wsId}/search` | searchWorkspace / suggest | member |

---

## 3. Endpoint Details

### 3.1 Search — `GET /workspaces/{wsId}/search`

**Query params:**

| Param | Values | Notes |
|---|---|---|
| `q` | string (1–200 after trim) | Required for meaningful results; empty → empty groups |
| `type` | `all` (default) · `issues` · `projects` · `cycles` · `members` | "Search within" dropdown |
| `status` | issue statuses | issues only |
| `priority` | issue priorities | issues only |
| `assigneeId` · `projectId` · `cycleId` | ids | issues only |
| `dueDateFrom` / `dueDateTo` | dates | issues only |
| `sort` | `relevance` (default) · `updatedAt` · `createdAt` | relevance = ts_rank, then updatedAt |
| `order` | `desc` (default) · `asc` | |
| `limit` | 20 (All) / 50 (type-filtered) | capped |

**Responses:**
- `200` — `{ q, total, results: { issues: IssueCard[], projects: ProjectCard[], cycles: CycleCard[], members: [{ id, name, role }] } }` — groups present only when selected via `type`; `total` = sum of returned (capped) rows
- `400` — `SEARCH_INVALID_INPUT` (q > 200 chars)
- `403` — `SEARCH_FORBIDDEN` (non-member)
- `429` — `SEARCH_RATE_LIMITED`

**Notes:**
- Suggestions use the same endpoint with `limit=5` (title/name-only rendering client-side).
- Archived entities are always excluded (query-time filter).
- Member results: name + role only — **email never included** (decision).

---

## 4. Error Codes (search domain)

| Code | Status | Meaning |
|---|---|---|
| `SEARCH_INVALID_INPUT` | 400 | Query too long / malformed params |
| `SEARCH_UNAUTHORIZED` | 401 | No valid session |
| `SEARCH_FORBIDDEN` | 403 | Not a workspace member |
| `SEARCH_RATE_LIMITED` | 429 | Rate limit hit |

---

## 5. Rate Limiting

| Endpoint | Limit |
|---|---|
| `GET /search` | **60/min per user** (debounced typing; headroom for suggestion burst) |

---

## 6. Web Integration (consumers)

| Web surface | Endpoint(s) |
|---|---|
| Header search box → suggestions overlay | `GET /search?q=…&limit=5` (debounced ~300ms) |
| Suggestions → full search (Enter / click) | `GET /search?q=…` (grouped results page/overlay) |
| "Search within" type dropdown | `type` param |
| Result filters (status/priority/assignee/dates) | Issue-only filter params |
| Result navigation | Client-side routing to the entity's page/drawer |

All calls go **through the Next proxy** (ADR-003).

---

## 7. OpenAPI & Shared Contracts

- Zod schemas in `packages/shared`: `SearchQuery` (strict — unknown params rejected), `SearchResponse`, `SearchGroup` — shared with web + OpenAPI generation.
- Result card shapes **reuse** the issues/projects/cycles/members contracts (no parallel types).

---

## 8. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Highlights/snippets | Post-MVP (advanced search) — confirm no snippet rendering needed in MVP results |
| 2 | Search within project/cycle pages | Scoped variant (`?projectId=` as a *constraint* from the page context) — trivial param reuse; confirm implementation |
| 3 | Member email in results | Excluded per decision — confirm privacy stance holds for the directory too (members list endpoint already exposes email to members — consistent?) |
