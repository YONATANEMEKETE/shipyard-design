# Projects — Feature Spec

**Status:** Approved
**Last updated:** 2026-08-22
**Design sources:** PRD §5.6 · §7.3 · UX User Flow (projects) · UI 03-UI
**Technical design:** Excluded by design — produced during this feature's implementation step, driven by this behavioral spec.

---

## 1. What this feature is about

Projects owns the **initiative**: a named container that groups related issues toward a shared objective, with an owner, progress, and a lifecycle of its own. Projects and Cycles are independent — any project↔cycle connection is derived through issues, never stored.

## 2. What users can do

- Create, view, edit, and delete projects (create/edit: Owner/Admin; view: all members).
- Set a project's operational status: Planned / Active / Completed (free switching).
- Assign a Project Owner and transfer ownership.
- Archive and restore a project.
- See a project's progress (derived from its issues).
- Group issues into a project (issue-level action).
- View projects as a list or Kanban board (per-user preference).
- Remove an issue from a project without deleting the issue.

## 3. Main behaviors & actions

### 3.1 Creation & fields
- Name is **required** and unique per workspace (trimmed, case-insensitive); archived projects reserve their name; deletion releases it.
- Optional: description, start date, target date.
- Creator becomes the Project Owner by default; ownership is **accountability, not authority** — it grants no permissions.

### 3.2 Status & lifecycle
- Operational status (Planned ⇄ Active ⇄ Completed) switches freely, any direction, no confirmation.
- **Archive** (Owner/Admin, confirmed): read-only; keeps its last operational status; never appears on boards.
- **Restore** (confirmed): returns to the stored operational status.
- **Delete** (Owner/Admin, confirmed, verbose warning): permanent. Issues are **not** deleted — they stay in the workspace, no longer assigned to the project, and the name is released.

### 3.3 Ownership
- Transfer (Owner/Admin) to any current workspace member; the recipient's workspace role is unchanged.
- When their member is removed or leaves, all owned projects transfer to the Workspace Owner (archived included) — fixed by rule, in the same operation.

### 3.4 Progress
- Derived from issues: completed vs total. Blocked issues do not affect completion calculations. Progress is never manually set.

### 3.5 Board semantics (same contract as issues)
- Columns = the three operational statuses.
- Drag = immediate status change, same semantics as the status control (no extra confirmation).
- Same-column drag does not reorder (results follow the active sort; no manual ranking in MVP).
- Empty columns remain visible as drop targets.
- Failed drag → card returns to previous column + error; concurrent change → card refreshes to latest state + notice.
- Archived is never a column; archived projects are list-only.

## 4. User flows (high level)

1. **Create:** projects → new project → name + optional details → created active, creator = Owner.
2. **Board:** drag card between Planned/Active/Completed columns → status updates immediately.
3. **Detail:** project page shows info, owner, progress, issue list, and actions (edit, transfer, archive, delete).
4. **Archive/restore:** confirm → hidden from boards/lists (except archived view) → restore returns to prior status.
5. **Delete:** warning with issue count → permanent; issues survive unassigned.

## 5. Business rules

1. Every project belongs to exactly one workspace.
2. Exactly one Project Owner per project, always a current workspace member.
3. Ownership grants no permissions.
4. Names unique per workspace (trim + case-insensitive); archived reserve; deleted releases.
5. Owner/Admin transfer a non-archived project to any current member.
6. On member removal/departure, owned projects transfer to the Workspace Owner automatically (archived included).
7. Operational statuses switch freely; Archived is a separate confirmed lifecycle state.
8. Restore returns to the stored operational status.
9. Delete is permanent and never deletes issues — it clears their project assignment as one operation.
10. Project↔cycle relationships are derived through issues; never stored.
11. Progress is derived from issue statuses; blocked issues do not count against completion.
12. View preference (List/Kanban) is per user per workspace, independent of the issues preference; list-only subviews never overwrite it.

## 6. Out of scope (MVP)

Templates, milestones, dependencies, roadmaps, health indicators, custom fields, cross-project reporting, budget/resource tracking.

## 7. Open product questions

| # | Question | Notes |
|---|---|---|
| 1 | Activity granularity | Action-level records like issues — confirm |
| 2 | List pagination | Projects are few — likely no pagination needed |
| 3 | Name normalization | Behavior is settled (unique per workspace, case-insensitive); mechanism decided at implementation |
