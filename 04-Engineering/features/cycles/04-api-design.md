# Cycles — API Design

**Module:** `apps/api/src/modules/cycles`
**Status:** Draft v0.1 — 2026-08-12
**Base URL:** `/api/v1` (proxied by Next.js; API internal-only)

---

## 1. Conventions

- **Resource:** `/api/v1/workspaces/{workspaceId}/cycles`.
- **Guards:** `requireSession → requireWorkspaceMember`; all lifecycle actions add `requireRole(OWNER | ADMIN)`.
- **Lifecycle actions are explicit endpoints** — there is no generic status field (controlled transitions, UX Decision 15).
- **Cycle issues** are listed via the issues endpoint (`GET /issues?cycleId=…`) — no duplicated nested list (DRY, same as projects).
- **Errors:** global envelope; codes in §4.

---

## 2. Endpoint Map

| Method | Path | Domain op | Guard |
|---|---|---|---|
| POST | `/workspaces/{wsId}/cycles` | createCycle | **Owner / Admin** |
| GET | `/workspaces/{wsId}/cycles` | listCycles | member |
| GET | `/workspaces/{wsId}/cycles/{cycleId}` | getCycle (+ progress, issues, activity) | member |
| PATCH | `/workspaces/{wsId}/cycles/{cycleId}` | updateCycle (Planned/Active only) | **Owner / Admin** |
| POST | `/workspaces/{wsId}/cycles/{cycleId}/start` | startCycle | **Owner / Admin** |
| POST | `/workspaces/{wsId}/cycles/{cycleId}/complete` | completeCycle | **Owner / Admin** |
| POST | `/workspaces/{wsId}/cycles/{cycleId}/reopen` | reopenCycle | **Owner / Admin** |
| POST | `/workspaces/{wsId}/cycles/{cycleId}/archive` | archiveCycle | **Owner / Admin** |
| POST | `/workspaces/{wsId}/cycles/{cycleId}/restore` | restoreCycle | **Owner / Admin** |
| DELETE | `/workspaces/{wsId}/cycles/{cycleId}` | deleteCycle (future Planned only) | **Owner / Admin** |

---

## 3. Endpoint Details

### 3.1 Create — `POST /workspaces/{wsId}/cycles`

**Body (Zod):** `{ name: string(1..64), startDate: date, endDate: date (>= startDate), goal?: string }`

**Responses:**
- `201` — `{ cycle: { id, name, startDate, endDate, status: "PLANNED", progress } }`
- `400` — `CYCLE_INVALID_INPUT` (incl. endDate < startDate)
- `409` — `CYCLE_NAME_CONFLICT` · `CYCLE_OVERLAP` (details: conflicting cycle)
- `403` — `CYCLE_FORBIDDEN` (member) · `429`

### 3.2 List — `GET /workspaces/{wsId}/cycles`

**Query params:** `status` (multi) · `startDateFrom/To` · `endDateFrom/To` · `sort` (`startDate` default · `createdAt` · `name`) · `order`.

**Responses:** `200` `{ cycles: CycleCard[] }` — `CycleCard` = name, goal, status, startDate, endDate, progress, archivedAt. Non-archived first (Active, then chronological), archived section last.

### 3.3 Get — `GET /workspaces/{wsId}/cycles/{cycleId}`

**Responses:** `200` `{ cycle: CycleDetail }` — full record + `progress: { total, completed, percent }` + `issues: IssueCard[]` + `activity: CycleActivity[]` · `404 CYCLE_NOT_FOUND` (or not a member — identical response).

### 3.4 Update — `PATCH /workspaces/{wsId}/cycles/{cycleId}`

**Body:** any subset of `{ name?, goal?, startDate?, endDate? }`.

**Business rules:** status must be PLANNED or ACTIVE (`409 CYCLE_STATE_CONFLICT` for Completed — read-only unless reopened; `409 CYCLE_ARCHIVED` for archived) · date edits re-check overlap excluding self (`409 CYCLE_OVERLAP`) · name change checks normalized conflict (`409 CYCLE_NAME_CONFLICT`).

**Responses:** `200` `{ cycle }` · `400` · `403` · `404` · `409` · `429`.

### 3.5 Lifecycle actions — `POST …/start | complete | reopen | archive | restore`

| Action | From → To | Specific guards |
|---|---|---|
| `start` | Planned → Active | no other ACTIVE → `409 CYCLE_ACTIVE_LIMIT` (details: conflicting cycle) |
| `complete` | Active → Completed | must be ACTIVE |
| `reopen` | Completed → Active | no other ACTIVE · no overlap with non-archived cycles |
| `archive` | Planned/Completed → Archived | must NOT be ACTIVE (`409` — complete first) |
| `restore` | Archived → stored status | restored range must not overlap non-archived cycles |

All: `200 { cycle }` · `403` · `404` · `409 CYCLE_STATE_CONFLICT` (wrong source state).

### 3.6 Delete — `DELETE /workspaces/{wsId}/cycles/{cycleId}`

**Guards:** must be PLANNED **and** `startDate > now` (everything else → `409 CYCLE_STATE_CONFLICT`).

**Responses:** `204` (issues auto-unassigned atomically) · `403` · `404` · `409`.

---

## 4. Error Codes (cycles domain)

| Code | Status | Meaning |
|---|---|---|
| `CYCLE_INVALID_INPUT` | 400 | Zod validation / degenerate date range |
| `CYCLE_UNAUTHORIZED` | 401 | No valid session |
| `CYCLE_FORBIDDEN` | 403 | Member attempting an Owner/Admin-only action |
| `CYCLE_NOT_FOUND` | 404 | Unknown id or not in workspace (identical response) |
| `CYCLE_NAME_CONFLICT` | 409 | Normalized name conflict (archived reserve names) |
| `CYCLE_OVERLAP` | 409 | Date range intersects a non-archived cycle (create/edit/reopen/restore) |
| `CYCLE_ACTIVE_LIMIT` | 409 | Another cycle is already ACTIVE (start/reopen) |
| `CYCLE_STATE_CONFLICT` | 409 | Action invalid for the current status (incl. delete of non-future-planned) |
| `CYCLE_ARCHIVED` | 409 | Write on an archived cycle |
| `CYCLE_RATE_LIMITED` | 429 | Rate limit hit |

---

## 5. Rate Limiting

| Endpoint | Limit |
|---|---|
| `POST /cycles` · `PATCH /cycles/{id}` · all lifecycle actions · `DELETE` | 30/min per user |
| `GET /cycles` · `GET /cycles/{id}` | 120/min per user |

---

## 6. Web Integration (consumers)

| Web surface | Endpoint(s) |
|---|---|
| Cycles page (Active/Planned/Completed/Archived sections) | `GET /cycles` |
| Cycle Details page | `GET /cycles/{id}` · `PATCH /cycles/{id}` |
| Cycle issues tab | `GET /issues?cycleId={id}` (issues endpoint) |
| Create Cycle modal (cycles page, global create — Owner/Admin) | `POST /cycles` |
| Start / Complete / Reopen (confirmed dialogs, state-appropriate only) | `start` · `complete` · `reopen` |
| Archive / Restore (confirmed dialogs) | `archive` · `restore` |
| Delete (future Planned only, confirmed dialog) | `DELETE /cycles/{id}` |
| Issue modal cycle picker | `PATCH /issues/{id}` `{ cycleId }` |
| Dashboard "Current cycle" card | `GET /cycles` (active) via dashboard composition |

All calls go **through the Next proxy** (ADR-003).

---

## 7. OpenAPI & Shared Contracts

- Zod schemas in `packages/shared`: `CreateCycleInput`, `UpdateCycleInput`, `CycleCard`, `CycleDetail`, `CycleActivity` — shared with web + OpenAPI generation.
- `CycleStatus` enum in `packages/shared`; date schemas enforce `endDate >= startDate` at the contract level.

---

## 8. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Past-dated Active cycles | Allowed per decision — confirm no auto-complete behavior needed at endDate (no cron in MVP; cycles complete manually) |
| 2 | Overlap error details | `details.conflictingCycle` included — confirm web renders it (name + dates) |
| 3 | Cycle issue ordering | Default sort for cycle issues = issue list default (`createdAt`) — confirm no custom ordering in MVP |

