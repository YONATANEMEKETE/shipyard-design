# Dashboard — Domain Model

**Module:** `apps/api/src/modules/dashboard`
**Status:** Draft v0.1 — 2026-08-12
**PRD source:** §5.4 Dashboard · Dashboard spec `2026-08-02-dashboard-focused-productivity-hub-design.md`

---

## 1. Overview & Scope

Dashboard owns the **personal productivity hub**: a per-user, per-workspace composed view — My Work, Current Cycle, Active Projects, Recent Activity. It is a **view module**: almost everything is derived from other modules' data.

**In scope:**
- The composed dashboard payload (all four panels)
- **Recently viewed issues** — the one piece of state the dashboard owns
- Activity feed aggregation

**Out of scope:**
- The panels' underlying entities → issues / projects / cycles / comments modules
- Custom dashboard layouts, widgets, drag-to-reorder, pinned widgets → post-MVP (PRD §5.4 future)
- Analytics, reports, workload charts → post-MVP

---

## 2. Domain Entities

### 2.1 Dashboard (a view, not an entity)

Composed per request — no dashboard table exists. Content is workspace-scoped, personalized by the session user:

| Panel | Content | Source |
|---|---|---|
| **My Work** | Assigned open issues · Created open issues · Recently viewed issues | issues + recently-viewed state |
| **Current Cycle** | The workspace's active cycle + progress donut + dates (null when none) | cycles |
| **Active Projects** | Active projects with progress bars | projects |
| **Recent Activity** | Workspace-wide recent events (issue status changes, comments, project/cycle actions) | activity tables (union) |

**Invariants:**
- The dashboard is **personal**: My Work reflects the session user; Recently viewed is per-user.
- Content is workspace-scoped (the active workspace via the route).
- Everything shown is **derived at read time** — no stored snapshots, no drift.
- Empty panels render designed empty states (empty-states doc), never errors.
- Clicking any item navigates to the entity (issue/project/cycle/member drawer).

### 2.2 Recently Viewed (the only owned state)

A per-user trail of viewed issues, used by My Work's "Recently viewed" and future recents everywhere.

| Attribute | Notes |
|---|---|
| `userId` + `issueId` | Unique pair (one row per issue per user — revisits refresh the timestamp) |
| `viewedAt` | Updated on every view |
| Cap | 50 per user — oldest pruned (bounded, cheap) |

**Recording:** reading an issue's detail (`GET /issues/{id}`) records the view — a cheap upsert side-effect on the read path, implemented by the issues module calling the dashboard's `recordView` (same request, no extra round-trip).

---

## 3. Domain Invariants

1. The dashboard is personal + workspace-scoped; content is permission-aware (member's own work + workspace-visible entities).
2. All panel data is derived at read time — no snapshots, no caching layer in MVP.
3. Recently viewed: one row per (user, issue); revisits bump the timestamp; capped at 50 with oldest-first pruning.
4. View recording never fails the issue read (best-effort side-effect; failure is logged, not surfaced).
5. Empty states are data, not errors.
6. The activity feed is bounded (e.g., latest 20 events) and chronological (newest first).

---

## 4. Domain Operations

| Operation | Description | Requires |
|---|---|---|
| `getDashboard` | Compose all four panels for the session user | member |
| `recordView` (internal) | Upsert + prune recently-viewed (called by issues GET) | internal |

---

## 5. Cross-Module Contracts

| Contract | Detail |
|---|---|
| **issues** | My Work queries (assigned/created, open only) + `recordView` call on issue detail reads |
| **cycles** | Active cycle + progress (null-safe) |
| **projects** | Active projects + progress |
| **activity** | Union read of `IssueActivity` + `ProjectActivity` + `Comment.createdAt` (bounded, newest first) |
| **web** | Dashboard loads on navigation (no polling — only notifications poll); panels navigate to their entities |

---

## 6. Trust Boundaries & Security Properties

1. Composition queries are scoped by `workspaceId` + membership; My Work additionally by `userId = session`.
2. Recently-viewed writes are recipient-scoped (only your own views recorded).
3. The dashboard is read-only from the client's perspective — no mutation surface.
4. Bounded result sets per panel (10–20 items) keep the composed request cheap.

---

## 7. Non-Goals (MVP)

Per PRD §5.4 future: customizable dashboard, widget management, analytics, reports, workload charts, cross-workspace dashboard.

---

## 8. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Recently viewed privacy | Personal by definition — visible only to the owner; confirm no workspace-visible variant |
| 2 | Activity feed scope | Workspace-wide (all members' events) vs follows-only — PRD implies workspace-wide; confirm |
| 3 | Panel item counts | Propose 10 per My Work group, 20 for activity — confirm |
