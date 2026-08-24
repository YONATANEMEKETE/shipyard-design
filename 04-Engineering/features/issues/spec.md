# Issues — Feature Spec

**Status:** Approved
**Last updated:** 2026-08-22
**Design sources:** PRD §5.5 · §7.4 · UX User Flow 9–11 · UI 03-UI
**Technical design:** Excluded by design — produced during this feature's implementation step, driven by this behavioral spec.

---

## 1. What this feature is about

Issues owns the **unit of work**: creation, workflow statuses, prioritization, assignment, blocking, labels, due dates, archiving, and the list/Kanban views over them. It is the core workflow of the product.

## 2. What users can do

- Create, edit, and delete issues (delete: Owner/Admin only).
- Work an issue through the fixed workflow: Backlog → Todo → In Progress → Done.
- Set priority, assignee, project, cycle, labels, and due date.
- Mark an issue blocked (with a reason) and unblock it.
- Filter and sort lists; toggle between List and Kanban views.
- Archive and restore issues.
- Search issues within the workspace.
- See an issue's history (status/assignment/planning changes).

## 3. Main behaviors & actions

### 3.1 Creation & identity
- Title is required (the only mandatory field); description optional.
- Each issue gets a **workspace-unique display identifier** (e.g. `SHIP-024`) — never reused, even after deletion.
- Optional fields on create: description, priority, assignee, project, cycle, labels, due date.
- New issues default to Backlog, No Priority, unblocked.
- Assignee/project/cycle/labels must be from the **current workspace** — cross-workspace values are rejected.
- If an assignee is set, the issue is visible in that member's notifications (reassignment to the same person is a no-op).

### 3.2 Workflow
- Fixed four statuses: `BACKLOG → TODO → IN_PROGRESS → DONE` — no custom statuses in the MVP.
- Status changes are free in any direction; every change is recorded in history.

### 3.3 Blocked
- **Blocked is orthogonal** — a flag (with optional reason), not a workflow status. A blocked issue stays in its column, visually marked; it does not affect progress calculations.
- Only unfinished issues can be blocked.
- Moving a blocked issue to **Done** clears the flag and reason (recorded in history); returning to an active status does **not** restore it.

### 3.4 Planning fields
- Priority: No Priority (default) / Urgent / High / Medium / Low.
- One assignee (workspace member), one project, one cycle; many labels.
- Due date: optional.
- Project deletion / cycle deletion clears that assignment from issues automatically (issues survive).

### 3.5 List & Kanban views
- Views: All Issues / My Issues / Backlog / Blocked / Archived (list-only for Backlog, Blocked, Archived).
- Filters (AND-combined), sorting, and text search; List ⇄ Kanban toggle keeps the same filters.
- View preference (List vs Kanban) is stored per user per workspace; list-only subviews never overwrite it.
- **Kanban:** columns = the four statuses. Drag = immediate status update, same semantics as the dropdown. Same-column drag does not reorder (cards follow the active sort). Empty columns remain drop targets. Failed drag → card returns to previous column + error; concurrent change → card refreshes to latest state + notice.
- Archived issues never appear on a board.

### 3.6 Archive / restore / delete
- **Archive:** read-only; disappears from active lists; appears in Archived (list-only).
- **Restore:** returns to its prior status + blocked state (both preserved while archived).
- **Delete (Owner/Admin only):** permanent, confirmed; comments and notifications disappear with it; the identifier is never reused.

### 3.7 Labels
- Workspace-scoped tags; name unique per workspace (trim + case-insensitive).
- Create, rename, recolor; deleting a label unlinks it from issues (issues untouched).

## 4. User flows (high level)

1. **Create:** quick-add or create dialog → title + optional fields → issue appears in Backlog; assignee notified.
2. **Work:** drag card across columns or use the status control — status changes immediately and is recorded.
3. **Block:** flag issue + optional reason → blocked badge; resolving it moves it to Done and clears the flag.
4. **Filter/search:** use filters or text query on the list → one view, same filters in List and Kanban.
5. **Archive/restore:** from the issue page → gone from active views → restore from Archived.
6. **Delete (Owner/Admin):** confirm in the issue page → gone permanently.

## 5. Business rules

1. Every issue belongs to exactly one workspace; projects/cycles/assignees/labels are workspace-scoped too.
2. An issue belongs to at most one project and at most one cycle; project↔cycle relationships are derived through issues.
3. Every issue has one creator, one status, and one priority value.
4. Display identifiers are unique per workspace and never reused.
5. New issues default to Backlog, No Priority, unblocked.
6. Only unfinished issues can be blocked; Done clears flag + reason; re-activating never restores it.
7. Blocked state does not affect project/cycle completion calculations.
8. Archived issues are read-only; restore preserves status + blocked state.
9. Deletion is permanent; comments/notifications die with the issue; identifier never reused.
10. Members create/edit/archive/restore; only Owners and Admins delete.
11. History records status, blocked, assignee, and planning changes.
12. List/Kanban preference is per user per workspace (issues and projects independently); list-only subviews never overwrite it.
13. Assignee changes notify the new assignee; reassigning to the same person is a no-op.

## 6. Out of scope (MVP)

Parent-child issues, dependencies, subtasks, custom issue types, recurring issues, templates, watchers, story points, rich text editing, file attachments.

## 7. Open product questions

| # | Question | Notes |
|---|---|---|
| 1 | Priority scale | Proposed: No Priority / Urgent / High / Medium / Low — confirm |
| 2 | Label creation permission | Proposed: any member can create labels — confirm |
| 3 | Identifier prefix | Proposed `SHIP-###` per workspace; prefix constant vs workspace-derived — confirm |
| 4 | List pagination | Cursor pagination proposed for lists — decide at implementation |
