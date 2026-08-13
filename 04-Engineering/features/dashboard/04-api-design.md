# Dashboard — API Design

**Module:** `apps/api/src/modules/dashboard`
**Status:** Draft v0.1 — 2026-08-12
**Base URL:** `/api/v1` (proxied by Next.js; API internal-only)

---

## 1. Conventions

- **Resource:** `/api/v1/workspaces/{workspaceId}/dashboard` — the composed payload.
- **Guards:** `requireSession → requireWorkspaceMember`.
- **Read-only surface:** one GET; view recording is an internal side-effect of the issues read path (no public endpoint).
- **Errors:** global envelope; codes in §4.

---

## 2. Endpoint Map

| Method | Path | Domain op | Guard |
|---|---|---|---|
| GET | `/workspaces/{wsId}/dashboard` | getDashboard | member |

---

## 3. Endpoint Details

### 3.1 Get dashboard — `GET /workspaces/{wsId}/dashboard`

**Responses:** `200` —

```json
{
  "myWork": {
    "assigned":       [IssueCard],   // assigned to me, open, limit 10
    "created":        [IssueCard],   // created by me, open, limit 10
    "recentlyViewed": [IssueCard]    // my recents, limit 10
  },
  "currentCycle": {                  // null when no active cycle
    "id": "...", "name": "...", "startDate": "...", "endDate": "...",
    "progress": { "total": 12, "completed": 7, "percent": 58 }
  } | null,
  "activeProjects": [{               // ACTIVE, non-archived, limit 10
    "id": "...", "name": "...", "owner": { "id": "...", "name": "..." },
    "progress": { "total": 20, "completed": 14, "percent": 70 },
    "targetDate": "..."
  }],
  "activity": [{                     // newest first, limit 20
    "id": "...", "kind": "ISSUE_STATUS" | "ISSUE_CREATED" | "COMMENT" |
                     "PROJECT_STATUS" | "CYCLE_STARTED" | "...",
    "actor": { "id": "...", "name": "..." },
    "issue": { "id": "...", "identifier": "SHIP-024" } | null,
    "detail": { "from": "TODO", "to": "IN_PROGRESS" } | null,
    "createdAt": "..."
  }],
  "serverTime": "ISO-8601"
}
```

- `403` — `DASHBOARD_FORBIDDEN` (non-member) · `429` — `DASHBOARD_RATE_LIMITED`
- Card shapes reuse the owning modules' contracts (`IssueCard`, project/cycle card shapes).

---

## 4. Error Codes (dashboard domain)

| Code | Status | Meaning |
|---|---|---|
| `DASHBOARD_UNAUTHORIZED` | 401 | No valid session |
| `DASHBOARD_FORBIDDEN` | 403 | Not a workspace member |
| `DASHBOARD_RATE_LIMITED` | 429 | Rate limit hit |

---

## 5. Rate Limiting

| Endpoint | Limit |
|---|---|
| `GET /dashboard` | 60/min per user (landing page + manual refreshes) |

---

## 6. Web Integration (consumers)

| Web surface | Endpoint(s) |
|---|---|
| Workspace landing page (post-login / switcher target) | `GET /dashboard` |
| My Work panel — Assigned / Created / Recently viewed | Rendered from `myWork`; items navigate to issues |
| Current Cycle donut card | `currentCycle` (null → empty state); "open cycle" CTA |
| Active Projects cards | `activeProjects`; navigate to project details |
| Recent Activity feed | `activity`; kind-based rendering; navigate to linked entities |
| Empty states | Per panel when arrays are empty (empty-states doc) |

All calls go **through the Next proxy** (ADR-003).

---

## 7. OpenAPI & Shared Contracts

- Zod schemas in `packages/shared`: `DashboardResponse`, `MyWorkGroup`, `ActivityEvent` — the dashboard's own types; card shapes imported from the owning modules.
- `serverTime` included so the web can render relative times ("2h ago") consistently.

---

## 8. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Activity kind enum completeness | MVP kinds: ISSUE_CREATED, ISSUE_STATUS, ISSUE_ASSIGNED, COMMENT, PROJECT_CREATED, PROJECT_STATUS, CYCLE_STARTED, CYCLE_COMPLETED — confirm nothing else needed |
| 2 | Refresh behavior | Load-on-navigation only — confirm no interval refetch for the dashboard (notifications poll remains the only poll) |
| 3 | Recently viewed across entity types | Issues only in MVP — projects/cycles recents post-MVP |
