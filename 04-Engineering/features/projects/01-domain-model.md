# Projects — Domain Model

**Module:** `apps/api/src/modules/projects`
**Status:** Draft v0.1 — 2026-08-12
**PRD source:** §5.6 Projects · §7.3 Project Rules

---

## 1. Overview & Scope

Projects owns the **initiative**: a named container that groups related issues toward a shared objective, with ownership, progress, and a lifecycle of its own.

**In scope:**
- Project CRUD + planning fields (description, start date, target date)
- The operational status model (Planned / Active / Completed) + the Archived lifecycle state
- Project ownership (exactly one owner; transfer; auto-transfer on member removal)
- Progress (derived from issues)
- List + Kanban views, per-user view preferences

**Out of scope:**
- Issues themselves → `issues` module (projects only reference them)
- Cycles → `cycles` module (**projects and cycles are independent** — any project↔cycle connection is derived through issues, never stored)
- Roles/membership mechanics → `members` (projects honor the auto-transfer contract)

---

## 2. Domain Entities

### 2.1 Project

| Attribute | Notes |
|---|---|
| `name` | Required; **unique per workspace** (trimmed, case-insensitive comparison). Archived projects reserve their name; deletion releases it. |
| `description` | Optional. |
| `status` | Operational: `PLANNED · ACTIVE · COMPLETED` — switchable freely, no confirmation. |
| `owner` | Exactly one **Project Owner** — a current workspace member; creator by default. **Ownership grants no permissions.** |
| `startDate` / `targetDate` | Optional dates (filterable). |
| `progress` | **Derived** from issues (completed / total) — never stored. |
| `archived` | Lifecycle state, **not** a status: only via confirmed Archive/Restore; read-only while archived. |

**Invariants:**
- Every project belongs to exactly one workspace.
- Project names are unique per workspace after trim + case-insensitive compare (different workspaces may share names).
- Archived projects reserve their name; permanent deletion releases it.
- Project ownership grants no additional workspace or resource permissions.
- Projects do not directly contain or own cycles — relationships are derived through issues.
- A project can hold zero or more issues; an issue belongs to at most one project.
- Project deletion is permanent, never deletes its issues, and clears their project assignment atomically.

### 2.2 Project Owner (role within the project)

- The creator is the initial owner.
- **Workspace Owners and Admins** may transfer a non-archived project to any other current workspace member (Owner, Admin, or Member) — the recipient's workspace role is unchanged.
- On member removal or departure: owned projects (archived included) transfer automatically to the **Workspace Owner** in the same operation (members contract — recipient fixed by rule, no choice).

---

## 3. Status & Lifecycle Model

```
OPERATIONAL (free switching, no confirmation):
   PLANNED ⇄ ACTIVE ⇄ COMPLETED        (any direction, any time)

LIFECYCLE (confirmed actions only):
   non-archived ──Archive──▶ ARCHIVED ──Restore──▶ stored operational status
        (read-only while archived; name stays reserved)
```

**Rules:**
- The status control never offers Archived — it is not an operational status.
- Archive stores the current operational status; restore returns to it.
- An archived project never appears on the Kanban board; Archived is never a column.
- Project status changes are recorded in project activity.

### 3.1 Kanban semantics

Same contract as issues (PRD symmetry): columns = Planned/Active/Completed; drag = immediate status update via the same endpoint as the control; same-column drag never reorders; empty columns remain drop targets; failed drag → revert + error; concurrent change → refresh + notice.

---

## 4. Domain Invariants

From PRD §7.3, condensed:

1. Every project belongs to exactly one workspace.
2. Every project has exactly one Project Owner, who is a current member of the workspace.
3. Project ownership grants no permissions (accountability, not authority).
4. Names: unique per workspace after trim + case-insensitive comparison; archived projects reserve names; deletion releases them.
5. Workspace Owners and Admins may transfer a non-archived project to any current member.
6. On member removal/departure, owned projects transfer automatically to the Workspace Owner (archived included).
7. Planned/Active/Completed switch freely; Archived is separate and confirmed; archived = read-only.
8. Restoring a project returns it to its stored operational status.
9. Deleting a project is permanent; issues remain, unassigned; deletion + unassignment are atomic.
10. Removing an issue from a project never deletes the issue.
11. Project↔cycle relationships are derived through issues — never stored.
12. Progress is derived from issue statuses; blocked issues do not affect completion calculations.
13. View preference: per user per workspace, independent of the issue view preference; list-only subviews never overwrite it.

---

## 5. Domain Operations

| Operation | Description | Requires |
|---|---|---|
| `createProject` | New project; creator = Project Owner; name uniqueness enforced | **Owner / Admin** |
| `getProject` | Detail: fields + owner + progress + issues summary + derived cycles | member |
| `listProjects` | Filtered list (status/owner/dates) — List or Kanban data | member |
| `updateProject` | Name/description/dates/status (free switch) | member (matrix: edit = all) |
| `transferOwnership` | Project Owner → any current member (non-archived only) | **Owner / Admin** |
| `archiveProject` / `restoreProject` | Confirmed lifecycle round-trip | **Owner / Admin** |
| `deleteProject` | Permanent; issues unassigned atomically; name released | **Owner / Admin** |
| `transferOwnedProjects` | Internal: auto-transfer on member removal/leave (called by members within its transaction) | members service |

---

## 6. Cross-Module Contracts

| Contract | Detail |
|---|---|
| **issues** | `Issue.projectId` FK with `onDelete: SetNull` — deletion unassigns atomically in the DB (issues data model §3) |
| **members** | Remove/leave calls `projectsService.transferOwnedProjects(userId → workspaceOwnerId)` inside the same transaction; archived projects included |
| **cycles** | No direct relation — project↔cycle visibility derived through shared issues |
| **workspace** | Cascade on workspace delete |
| **dashboard / search** | Progress + status aggregates read through the project repository (query-only) |
| **notifications** | Post-MVP: project ownership change notifications (PRD future list) — no schema change needed |

---

## 7. Trust Boundaries & Security Properties

1. Create/transfer/archive/restore/delete pass `requireRole(OWNER | ADMIN)`; edits pass member (PRD matrix).
2. Transfer target is validated as a **current workspace member** server-side (re-verified inside the transaction — TOCTOU-safe, same pattern as ownership transfer in members).
3. Names are normalized server-side (trim + lowercase compare); the DB unique is the backstop.
4. Archived projects reject writes at the data layer.
5. Progress is computed, never client-supplied.
6. Deletion's dialog warns about issue count — but the server enforces the atomicity (SetNull in the same DELETE).

---

## 8. Non-Goals (MVP)

Per PRD §5.6 future enhancements: project templates, milestones, dependencies, roadmaps, health indicators, custom fields, cross-project reporting, budget/resource tracking.

---

## 9. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Project activity granularity | Same model as issues (action enum + Json detail) — confirm |
| 2 | List pagination | Projects are fewer than issues — propose no pagination in MVP (note for future) |
| 3 | Name normalization storage | Propose `nameNormalized` column (lowercase, trimmed) + unique(workspaceId, nameNormalized) — the DB then enforces case-insensitive uniqueness directly |
