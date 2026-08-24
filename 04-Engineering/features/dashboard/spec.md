# Dashboard — Feature Spec

**Status:** Approved
**Last updated:** 2026-08-22
**Design sources:** PRD §5.4 · UI dashboard design (`dashboard-focused-productivity-hub`)
**Technical design:** Excluded by design — produced during this feature's implementation step, driven by this behavioral spec.

---

## 1. What this feature is about

Dashboard is the **personal productivity hub**: a per-user, per-workspace composed landing view — My Work, Current Cycle, Active Projects, and Recent Activity. It is a **view module**: everything is derived from other features' data at read time (no stored snapshots, no drift).

## 2. What users can do

- Land on a composed overview of their workspace instead of navigating feature by feature.
- See **My Work**: issues assigned to them (open), issues they created (open), and their recently viewed issues.
- See the **Current Cycle** (the workspace's active cycle + progress + dates; empty state when none is running).
- See **Active Projects** with progress.
- See **Recent Activity**: workspace-wide recent events (status changes, comments, project/cycle actions).
- Click any item to navigate to the underlying issue, project, cycle, or member.
- See designed empty states everywhere — no panel errors out when there's no data.

## 3. Main behaviors & actions

### 3.1 Panels
| Panel | Content | Notes |
|---|---|---|
| **My Work** | Assigned open issues · created open issues · recently viewed issues | Personal — reflects the signed-in user |
| **Current Cycle** | Active cycle + progress + dates | Empty state when none active |
| **Active Projects** | Active projects with progress bars | — |
| **Recent Activity** | Latest workspace events, newest first | Bounded list (e.g., 20) |

### 3.2 Recently viewed
- A personal trail of issues the user has opened (issue detail views).
- Revisiting an issue bumps it to the top; the trail is capped (~50), oldest pruned.
- Personal: only the owner sees their own recently-viewed trail.

### 3.3 Behavior rules
- Everything is derived at read time — the dashboard never stores panel content.
- The dashboard is workspace-scoped and permission-aware (member's own work + workspace-visible entities).
- The dashboard loads on navigation; no auto-refresh is required (only the notifications badge polls).
- Viewing an issue's detail records it as recently viewed — a best-effort side effect that never blocks the read.

## 4. User flows (high level)

1. **Enter:** sign in → dashboard → four panels render (or designed empty states).
2. **Drill down:** click a My Work item → issue detail; click current cycle → cycle page; click project → project page.
3. **Return:** revisit the dashboard → recently viewed shows the issues just opened, most recent first.

## 5. Business rules

1. The dashboard is personal + workspace-scoped; content is permission-aware.
2. All panel data is derived at read time — no snapshots or caching layers in MVP.
3. Recently viewed: one entry per (user, issue); revisits bump; capped with oldest-first pruning.
4. View recording never fails the issue read (best-effort side effect).
5. Empty states are data, not errors.
6. The activity feed is bounded and newest-first.

## 6. Out of scope (MVP)

Customizable layouts, widget management, drag-to-reorder, pinned widgets, analytics, reports, workload charts, cross-workspace dashboard.

## 7. Open product questions

| # | Question | Notes |
|---|---|---|
| 1 | Recently viewed privacy | Personal only — confirm no workspace-visible variant |
| 2 | Activity feed scope | Workspace-wide (all members' events) — confirm |
| 3 | Panel item counts | ~10 per My Work group, 20 for activity — confirm |
