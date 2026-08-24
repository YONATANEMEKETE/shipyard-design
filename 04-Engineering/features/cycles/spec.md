# Cycles — Feature Spec

**Status:** Approved
**Last updated:** 2026-08-22
**Design sources:** PRD §5.7 · §7.5 · UX Decision 15 · UX User Flow (cycles)
**Technical design:** Excluded by design — produced during this feature's implementation step, driven by this behavioral spec.

---

## 1. What this feature is about

Cycles owns the **time-boxed iteration**: a fixed development period (date range) with a controlled lifecycle — Start / Complete / Reopen / Archive / Restore / Delete — and a strict scheduling contract: no overlapping cycles, at most one active cycle per workspace. Progress is tracked through issues.

## 2. What users can do

- Create, view, edit, and delete cycles (manage: Owner/Admin; view: all members).
- Set a cycle name, goal, and date range (both dates required).
- Start / complete / reopen a cycle through explicit lifecycle actions.
- Archive and restore cycles.
- Assign issues to a cycle (issue-level action) and filter issues by cycle.
- See a cycle's issue list and progress.
- See which cycle is currently active (and in the Dashboard).

## 3. Main behaviors & actions

### 3.1 Scheduling
- Non-archived cycles **never overlap** (dates are inclusive; the next cycle starts after the previous ends).
- At most **one cycle is Active** per workspace at a time.
- Names are unique per workspace (trim + case-insensitive); archived cycles reserve names; deletion releases.
- Editing dates keeps the no-overlap rule.

### 3.2 Lifecycle (controlled transitions only — no generic status control)

| From | To | Action | Guards |
|---|---|---|---|
| Planned | Active | **Start** | no other Active cycle |
| Active | Completed | **Complete** | — |
| Completed | Active | **Reopen** | no other Active · no date conflict |
| Planned / Completed | Archived | **Archive** | Active must be completed first |
| Archived | stored status | **Restore** | restored range must not overlap a non-archived cycle |
| Planned (future) | — | **Delete** | future Planned only; issues unassigned |

- The UI exposes only actions valid for the current state and explains blocked transitions (e.g., "complete the active cycle first").
- Active cycles can't be archived directly — complete first.
- Completed cycles are read-only unless reopened.

### 3.3 Issues & progress
- An issue belongs to at most one cycle.
- Adding/removing an issue to/from a cycle is an issue-level action.
- Completing a cycle does **not** complete its unfinished issues.
- Deleting a future Planned cycle unassigns its issues (never deletes them).
- Progress is derived from issues (completed / total); blocked issues do not affect it.
- Archived cycles remain for historical reference and don't block scheduling.

## 4. User flows (high level)

1. **Create:** cycles → new cycle → name + goal + dates → Planned.
2. **Start:** Planned → Active (fails with a clear explanation if another cycle is active or dates conflict).
3. **Complete:** Active → Completed; unfinished issues stay open.
4. **Reopen:** Completed → Active (same guards).
5. **Archive/restore:** planned or completed → archived; restore returns to the previous status (blocked if the range now conflicts).
6. **Delete:** a future planned cycle → confirmed → issues stay, unassigned.

## 5. Business rules

1. Every cycle belongs to exactly one workspace; an issue belongs to at most one cycle.
2. Names unique per workspace (trim + case-insensitive); archived reserve; deletion releases.
3. Non-archived cycle date ranges never overlap (inclusive dates); edits preserve this.
4. Only one cycle may be Active per workspace.
5. Lifecycle changes happen only via controlled actions (Start/Complete/Reopen/Archive/Restore/Delete).
6. An Active cycle cannot be archived — complete it first.
7. Completed cycles are read-only unless reopened (reopen respects active-limit + no-overlap).
8. Restore returns to the stored pre-archive status and must not conflict with non-archived cycles.
9. Completing a cycle does not complete unfinished issues.
10. Deleting a future Planned cycle unassigns its issues without deleting them.
11. Blocked issues don't affect completion calculations.
12. Cycles don't belong to projects; project↔cycle relationships are derived through issues.

## 6. Out of scope (MVP)

Recurring cycles, carry-over of unfinished issues, velocity reporting, burndown charts, capacity planning, workload balancing.

## 7. Open product questions

| # | Question | Notes |
|---|---|---|
| 1 | Cycle length guardrails | None proposed (any valid non-overlapping range) |
| 2 | Start date in the past | Allowed (cycles can start late) |
| 3 | Start conflict UX | Show the conflicting active cycle when Start is blocked |
