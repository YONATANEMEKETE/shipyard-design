# Cycles — Domain Model

**Module:** `apps/api/src/modules/cycles`
**Status:** Draft v0.1 — 2026-08-12
**PRD source:** §5.7 Cycles · §7.5 Cycle Rules · UX Decision 15

---

## 1. Overview & Scope

Cycles owns the **time-boxed iteration**: fixed development periods with a controlled lifecycle, a strict scheduling contract (no overlaps, one active), and progress tracking.

**In scope:**
- Cycle CRUD + goal/description + date range
- The controlled lifecycle: Start / Complete / Reopen / Archive / Restore / Delete
- Scheduling rules: non-overlapping ranges, one active cycle
- Progress (derived from issues)

**Out of scope:**
- Issues → `issues` module (a cycle references issues; membership is `Issue.cycleId`, owned there)
- Projects → `projects` module (**no direct relation** — project↔cycle connections derived through issues)
- Recurring cycles, burndown/velocity charts, capacity planning → post-MVP (PRD §5.7 future)

---

## 2. Domain Entities

### 2.1 Cycle

| Attribute | Notes |
|---|---|
| `name` | Required; unique per workspace (trim + case-insensitive); archived cycles reserve names; delete releases |
| `goal` | Optional description of the iteration's objective |
| `startDate` / `endDate` | **Both required**; inclusive range; must not overlap any other non-archived cycle |
| `status` | `PLANNED · ACTIVE · COMPLETED` (+ `ARCHIVED` lifecycle state) |
| `progress` | Derived: completed / total issues |

**Invariants:**
- Every cycle belongs to exactly one workspace.
- Non-archived cycle date ranges never overlap (inclusive — a following cycle starts *after* the preceding one ends).
- At most one cycle is Active per workspace.
- An issue belongs to at most one cycle.
- Completed cycles are read-only unless reopened.
- Completing a cycle does **not** complete its unfinished issues.
- Archived cycles remain for historical reference; they do not block scheduling — but **restoring** one must not overlap a non-archived cycle.
- Deleting a future Planned cycle unassigns its issues (never deletes them).

---

## 3. Lifecycle State Machine (controlled transitions)

```
        Start (no other active, no overlap)
PLANNED ──────────────────────────────────▶ ACTIVE
   ▲                                          │
   │        Reopen (no other active,          │ Complete
   │         no date conflict)                ▼
   └────────────────────────────────────── COMPLETED
   │                                            │
   │  Archive (Planned or Completed only)       │ (Active must complete FIRST)
   ▼                                            ▼
ARCHIVED ◀──────────────────────────────────────┘
   │
   └── Restore → stored pre-archive status (Planned or Completed)
       (blocked if restored range overlaps a non-archived cycle)
```

**The transition table (PRD §5.7):**

| From | To | Action | Guards |
|---|---|---|---|
| Planned | Active | **Start** | no other Active cycle in workspace |
| Active | Completed | **Complete** | — |
| Completed | Active | **Reopen** | no other Active · no date conflict with non-archived cycles |
| Planned / Completed | Archived | **Archive** | Active must be completed first |
| Archived | stored status | **Restore** | restored range must not overlap a non-archived cycle |
| Planned (future) | — | **Delete** | only future Planned cycles; issues unassigned |

**Rules:**
- Status is changed **only** through these actions — there is no generic status control (UX Decision 15).
- The UI exposes only actions valid for the cycle's current state and explains blocked transitions (e.g., "complete the active cycle first").

---

## 4. Domain Invariants

From PRD §7.5, condensed:

1. Every cycle belongs to exactly one workspace; issue-to-cycle membership is at most one per issue.
2. Cycle names unique per workspace after trim + case-insensitive comparison; archived reserve names; delete releases.
3. Non-archived cycle date ranges never overlap (inclusive dates); editing dates preserves this rule.
4. Only one cycle may be Active per workspace.
5. Lifecycle changes use the controlled actions only (Start/Complete/Reopen/Archive/Restore/Delete).
6. An Active cycle cannot be archived — it must be completed first.
7. Completed cycles are read-only unless reopened; reopening respects active-limit and no-overlap.
8. Restoring an archived cycle returns it to its stored pre-archive status and must not overlap non-archived cycles.
9. Completing a cycle does not automatically complete unfinished issues.
10. Deleting a future Planned cycle removes it and unassigns its issues without deleting them.
11. Blocked issues do not affect cycle completion calculations (progress is issue-status-derived).
12. Cycles do not directly belong to projects; project↔cycle relationships are derived through issues.

---

## 5. Domain Operations

| Operation | Description | Requires |
|---|---|---|
| `createCycle` | New Planned cycle; name + non-overlapping range enforced | **Owner / Admin** |
| `updateCycle` | Name/goal/dates (Planned or Active only; edits keep no-overlap) | **Owner / Admin** |
| `startCycle` | Planned → Active (one-active guard) | **Owner / Admin** |
| `completeCycle` | Active → Completed (read-only) | **Owner / Admin** |
| `reopenCycle` | Completed → Active (guards: active-limit + no-overlap) | **Owner / Admin** |
| `archiveCycle` | Planned/Completed → Archived | **Owner / Admin** |
| `restoreCycle` | Archived → stored status (no-overlap guard) | **Owner / Admin** |
| `deleteCycle` | Future Planned only; issues unassigned | **Owner / Admin** |
| `listCycles` / `getCycle` | Sections (Active/Planned/Completed/Archived) + detail with issues/progress | member |

*Adding/removing issues to a cycle is an **issue** operation (`PATCH /issues/{id} { cycleId }`) — cycleId is owned by the issues module; the cycle service only validates the target cycle.*

---

## 6. Cross-Module Contracts

| Contract | Detail |
|---|---|
| **issues** | `Issue.cycleId` FK with `onDelete: SetNull` — cycle deletion unassigns atomically; membership changes flow through the issues endpoint |
| **projects** | None direct — derived visibility only |
| **workspace** | Cascade on workspace delete |
| **dashboard** | "Current cycle" widget reads via cycle repository (active cycle + progress) |

---

## 7. Trust Boundaries & Security Properties

1. All lifecycle actions pass `requireRole(OWNER | ADMIN)` (matrix).
2. Scheduling rules are enforced at **two layers**: the service (friendly errors) and the database (exclusion constraint + partial unique index — see data model §4). The DB is the final referee under concurrency.
3. Date ranges are validated as inclusive and non-degenerate (end ≥ start).
4. Archived cycles reject writes at the data layer.
5. Progress is computed, never client-supplied.

---

## 8. Non-Goals (MVP)

Per PRD §5.7 future: recurring cycles, carry-over of unfinished issues, sprint velocity reporting, burndown charts, capacity planning, workload balancing.

---

## 9. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Cycle length guardrails | No min/max in PRD — propose none (any valid non-overlapping range) |
| 2 | Start date in the past | Allow (cycles can start late) — propose yes |
| 3 | "Today" conflicts with active cycle | Start guard returns `CYCLE_ACTIVE_LIMIT` with explanation (web shows the conflicting cycle) — confirm |
