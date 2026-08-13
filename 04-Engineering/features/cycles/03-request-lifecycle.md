# Cycles — Request Lifecycle

**Module:** `apps/api/src/modules/cycles`
**Status:** Draft v0.1 — 2026-08-12
**Relies on:** workspace lifecycle §5 (guard chain) · `02-data-model.md` · `04-api-design.md`

---

## 1. Overview

Cycles are **schedule-first**: every write touches the no-overlap + one-active contract, and every status change is a **controlled action** (never a generic status control). The DB (exclusion constraint + partial unique index) is the final referee; the service produces the friendly errors. Guard chain: `requireSession → requireWorkspaceMember`, with `requireRole(OWNER | ADMIN)` on every lifecycle action.

---

## 2. Flow — Create cycle

```
1. POST /workspaces/{wsId}/cycles
   body: { name, startDate, endDate, goal? }        [Zod: name 1-64, dates required,
                                                      endDate >= startDate (inclusive)]
2. guards: requireRole(OWNER | ADMIN)
3. name normalization conflict check                [409 CYCLE_NAME_CONFLICT]
4. overlap check (service, friendly):
   any non-archived cycle where [startDate, endDate] intersects
   → 409 CYCLE_OVERLAP (details: conflicting cycle name)
   (the exclusion constraint is the race-proof backstop — a P2004 maps here)
5. INSERT Cycle (status PLANNED) + CycleActivity (CREATED) — one transaction
6. 201 → { cycle } → web shows it under Planned
```

## 3. Flow — Update cycle (name / goal / dates)

```
1. PATCH /workspaces/{wsId}/cycles/{cycleId}
   body: { name?, goal?, startDate?, endDate? }     [Zod]
2. guards: requireRole(OWNER | ADMIN)
3. state guard: status must be PLANNED or ACTIVE
   (Completed → 409 CYCLE_STATE_CONFLICT — read-only unless reopened;
    Archived → 409 CYCLE_ARCHIVED)
4. name change → normalized conflict check           [409 CYCLE_NAME_CONFLICT]
5. date change → overlap re-check against OTHER non-archived cycles
   (excludes self)                                   [409 CYCLE_OVERLAP]
6. UPDATE + CycleActivity (UPDATED {from,to}) — one transaction
7. 200 → { cycle }
```

## 4. Flow — Start / Complete / Reopen

```
START (Planned → Active):
1. POST /workspaces/{wsId}/cycles/{cycleId}/start
2. guards: requireRole(OWNER | ADMIN)
3. must be PLANNED                                  [409 CYCLE_STATE_CONFLICT]
4. another cycle ACTIVE in workspace?
   → 409 CYCLE_ACTIVE_LIMIT (details: the active cycle's name — web explains)
   (partial unique index is the backstop under concurrent starts)
5. UPDATE status = ACTIVE + activity (STARTED)

COMPLETE (Active → Completed):
1. guards; must be ACTIVE                           [409]
2. UPDATE status = COMPLETED + activity (COMPLETED)
   — issues are NOT touched (domain rule 9); progress freezes by derivation
   — cycle is now read-only (update guard, flow 3)

REOPEN (Completed → Active):
1. guards; must be COMPLETED                        [409]
2. guards: no other ACTIVE cycle → 409 CYCLE_ACTIVE_LIMIT
3. guards: [startDate, endDate] must not overlap any non-archived cycle
   → 409 CYCLE_OVERLAP (dates conflict after completion — web suggests
   editing dates first or archiving the old cycle)
4. UPDATE status = ACTIVE + activity (REOPENED)
```

## 5. Flow — Archive / Restore

```
ARCHIVE (Planned or Completed only):
1. POST /workspaces/{wsId}/cycles/{cycleId}/archive
2. guards: requireRole(OWNER | ADMIN)
3. must be PLANNED or COMPLETED
   (ACTIVE → 409 CYCLE_STATE_CONFLICT — "complete the active cycle first",
    UX Decision 15; the web shows the explanation)
4. UPDATE archivedAt = now (status column preserved = restore target)
   + activity (ARCHIVED)
   — the cycle leaves the no-overlap constraint scope: history never blocks
     new scheduling

RESTORE (Archived → stored status):
1. POST /workspaces/{wsId}/cycles/{cycleId}/restore
2. guards; must be ARCHIVED                         [409]
3. overlap re-check: restored [startDate, endDate] vs non-archived cycles
   → 409 CYCLE_OVERLAP (re-enters the constraint scope — DB is backstop)
4. UPDATE archivedAt = null + activity (RESTORED)
   → returns to its STORED pre-archive status (Planned or Completed)
```

## 6. Flow — Delete cycle (future Planned only)

```
1. DELETE /workspaces/{wsId}/cycles/{cycleId}
2. guards: requireRole(OWNER | ADMIN)
3. must be PLANNED AND startDate in the future
   (anything else → 409 CYCLE_STATE_CONFLICT — only future planned cycles
    are deletable, PRD)
4. confirm dialog (web): "issues will be unassigned, not deleted"
5. DELETE Cycle
   → Issue.cycleId auto-SetNull in the SAME DB statement (issues data
     model §3) — atomic
   → activities cascade; name released
6. 204
```

## 7. Flow — Assigning issues to a cycle

Issue membership is an **issue** operation (the field lives on Issue):

```
PATCH /issues/{id} { cycleId }    → issue endpoint, member-level
  → the cycle service validates the target (exists, non-archived,
    non-completed? — assignable to any non-archived cycle; completed cycles
    are read-only but historical assignments remain)
  → IssueActivity records the change (PLANNING_CHANGED)
  → cycle progress updates by derivation (no write to Cycle)
```

## 8. Flow — List / detail

```
GET /workspaces/{wsId}/cycles?status=&startDateFrom=&startDateTo=
    &endDateFrom=&endDateTo=
  → 200 { cycles } — chronological sections (Active / Planned / Completed /
    Archived); list-only (cycles are never a Kanban workflow, PRD)

GET /workspaces/{wsId}/cycles/{cycleId}
  → 200 { cycle } + progress { total, completed, percent } + issues:
    IssueCard[] + activity: CycleActivity[]
    (issues fetched with the same card shape as the issues module)
```

## 9. Edge Cases & Failure Handling

| Case | Behavior |
|---|---|
| Two starts racing | Partial unique index rejects the second → 409 `CYCLE_ACTIVE_LIMIT` |
| Two overlapping creates racing | Exclusion constraint rejects → 409 `CYCLE_OVERLAP` |
| Create with endDate < startDate | 400 `CYCLE_INVALID_INPUT` (inclusive range, degenerate rejected) |
| Date edit that collides | 409 `CYCLE_OVERLAP` — edit rolled back, previous dates intact |
| Start a cycle whose dates already passed | Allowed (decision: no guardrail); completes normally |
| Reopen when dates now collide | 409 `CYCLE_OVERLAP` — web suggests editing dates or archiving |
| Complete with unfinished issues | Allowed — issues keep their statuses (domain rule 9) |
| Delete an active/completed/archived cycle | 409 `CYCLE_STATE_CONFLICT` |
| Delete a Planned cycle with a past start date | 409 (not "future") |
| Issue moved out of an active cycle | Fine — cycle progress recalculates by derivation |
| Archive an active cycle | 409 — must complete first (web explains) |

## 10. Dev vs Prod Differences

| Concern | Local dev | Production |
|---|---|---|
| `btree_gist` + exclusion constraint | Same migrations (local Postgres) | Neon supports both — same SQL |
| Everything else | Same behavior | Same |

