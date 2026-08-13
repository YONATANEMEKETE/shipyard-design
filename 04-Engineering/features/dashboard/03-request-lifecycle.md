# Dashboard — Request Lifecycle

**Module:** `apps/api/src/modules/dashboard`
**Status:** Draft v0.1 — 2026-08-12
**Relies on:** workspace lifecycle §5 (guard chain) · `02-data-model.md` · `04-api-design.md`

---

## 1. Overview

One composed read + one internal side-effect (view recording). No mutations, no polling — the dashboard loads on navigation and reflects current data every time.

Guard chain: `requireSession → requireWorkspaceMember`.

---

## 2. Flow — Compose dashboard

```
1. GET /workspaces/{wsId}/dashboard
2. requireSession → requireWorkspaceMember
3. PARALLEL composition (Promise.all — ~6 bounded queries):
   ├── myWork.assigned      → issues assigned to me, open, limit 10
   ├── myWork.created       → issues I created, open, limit 10
   ├── myWork.recentlyViewed→ RecentlyViewed JOIN Issue (mine), limit 10
   ├── currentCycle         → active cycle + progress (null when none)
   ├── activeProjects       → ACTIVE projects + progress, limit 10
   └── activity             → UNION IssueActivity + ProjectActivity + Comment,
                              limit 20, newest first
4. Assemble + shape:
   { myWork, currentCycle, activeProjects, activity, serverTime }
5. 200 → web renders the four panels (empty panels → designed empty states)
```

**Why parallel:** the panels are independent; sequential composition would add 5× latency to the workspace's landing page. Each query is bounded — worst case the dashboard costs ~6 small index scans.

## 3. Flow — View recording (side-effect on issue reads)

```
GET /workspaces/{wsId}/issues/{issueId}   (issues module — read path)
  → issues service calls dashboardService.recordView(userId, issueId)
  → BEST-EFFORT upsert: INSERT ... ON CONFLICT (userId, issueId)
    DO UPDATE SET viewedAt = now
  → if this was a NEW unique view: prune to 50 (oldest deleted)
  → failure is caught + logged — NEVER fails the issue read
```

**Why best-effort:** a "recently viewed" trail must never break the primary read path. The cap keeps the table bounded even under view spam.

## 4. Edge Cases & Failure Handling

| Case | Behavior |
|---|---|
| No active cycle | `currentCycle: null` → panel renders empty state ("no cycle yet") |
| No open issues assigned to me | Empty My Work group with empty-state CTA |
| Workspace with no projects | Active projects panel empty state |
| 51st unique issue viewed | Oldest pruned — cap holds at 50 |
| View recording fails (DB hiccup) | Logged, read succeeds — recents just don't update |
| Issue deleted after being viewed | Cascade removes it from recents |
| Activity events for deleted issues | Cascaded away before composition — no dead links |
| User navigates dashboard rapidly | No state, no cache — each load is fresh; rate limit 60/min |
| Member of 2 workspaces | Dashboard is per-workspace — each workspace's route composes its own |

## 5. Dev vs Prod Differences

| Concern | Local dev | Production |
|---|---|---|
| Everything | Same behavior | Same (pure aggregation — no external services) |
