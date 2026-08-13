# Issues — Domain Model

**Module:** `apps/api/src/modules/issues`
**Status:** Draft v0.1 — 2026-08-12
**PRD source:** §5.5 Issues · §7.4 Issue Rules

---

## 1. Overview & Scope

Issues owns the **unit of work**: creation, workflow statuses, prioritization, assignment, blocking, labels, due dates, archival, and the list/kanban views over them.

**In scope:**
- Issue CRUD + all planning fields (project, cycle, assignee, labels, due date, priority)
- The workflow (Backlog → Todo → In Progress → Done) + the **Blocked flag**
- Archive/restore (read-only while archived)
- List + Kanban views, per-user view preferences
- Issue identity (workspace-scoped identifier) and history

**Out of scope:**
- Projects/cycles *as entities* → their modules (issues only reference them)
- Comments → `comments` module (issue is the anchor)
- Notifications → `notifications` module (issues *emit* assignment events)
- Search infrastructure → `search` module (the tsvector index is defined here; query orchestration lives there)

---

## 2. Domain Entities

### 2.1 Issue

| Attribute | Notes |
|---|---|
| `identifier` | **Unique within a workspace** — display id, e.g. `SHIP-024` (design system sample). Sequence per workspace; never reused. |
| `title` | Required; the only mandatory field. |
| `description` | Optional. |
| `status` | `BACKLOG → TODO → IN_PROGRESS → DONE` — fixed workflow, no custom statuses in MVP. |
| `priority` | `NO_PRIORITY` (default) · `URGENT` · `HIGH` · `MEDIUM` · `LOW` — Linear-style scale (see open question 1). |
| `assignee` | 0..1 workspace member. |
| `project` | 0..1 — projects are independent entities (derived relationships only). |
| `cycle` | 0..1 — same independence rule. |
| `labels` | 0..n workspace labels. |
| `dueDate` | Optional date. |
| `blocked` | **Independent flag** + optional reason — NOT a workflow status. |
| `creator` | The creating member; immutable. |
| `history` | Status/blocked/planning changes recorded. |

**Invariants:**
- Every issue belongs to exactly one workspace.
- Every issue has exactly one status and one priority value.
- New issues: `BACKLOG` + `NO_PRIORITY` + unblocked, unless chosen otherwise.
- Only `BACKLOG`/`TODO`/`IN_PROGRESS` issues can be marked blocked.
- Moving a blocked issue to `DONE` clears its flag + reason; returning to an active status does **not** restore it.
- Identifiers are unique within a workspace and never reused (even after deletion).
- Archived issues are read-only; restoration returns them to their prior workflow status and blocked state (both are preserved as-is while archived — no snapshot needed).

### 2.2 Label

Workspace-scoped tag (name unique per workspace after trim + case-insensitive compare, same normalization rule as projects/cycles). Created by members (see open question 2); applied to issues via the join.

---

## 3. Workflow Model

```
                    ┌──────────┐
   create ─────────▶│ BACKLOG  │
                    └────┬─────┘
                         │ (moved to)
                    ┌────▼─────┐      ┌──────────┐
                    │   TODO   │─────▶│ IN PROGRESS│
                    └────┬─────┘      └────┬─────┘
                         │                 │
                    ┌────▼─────────────────▼─────┐
                    │           DONE             │
                    └────────────────────────────┘
```

- **Fixed four states** — no custom columns (PRD: custom workflows are post-MVP).
- Status changes are **free in any direction** (no transition restrictions beyond blocked rules) and recorded in history.
- **Blocked is orthogonal:** any unfinished issue carries a flag + optional reason; it does not affect status transitions, progress calculations, or Kanban columns (a blocked issue stays in its column, visually marked).

### 3.1 Kanban semantics (domain-level)

- All Issues / My Issues expose List ⇄ Kanban; columns = the four statuses.
- **Drag = immediate status update** (same rules as the dropdown), no confirmation.
- **Same-column drag does not reorder** — cards follow the active sort (no manual ranking in MVP).
- Empty columns remain visible as drop targets.
- Failed drag → card returns to previous column + error; concurrent change → card refreshes to latest state + notice.
- Backlog, Blocked, and Archived views are **list-only**; archived issues never appear on a board.

---

## 4. Domain Invariants

From PRD §5.5 business rules + §7.4, condensed:

1. Every issue belongs to exactly one workspace; projects/cycles/assignees/labels are workspace-scoped too.
2. An issue may belong to at most one project and at most one cycle (project↔cycle relationships are derived through issues — never stored on either).
3. Every issue has one creator, one status, one priority value.
4. Identifiers are unique per workspace and never reused.
5. New issues default to `BACKLOG`, `NO_PRIORITY`, unblocked.
6. Only unfinished issues may be blocked; `DONE` clears the flag and reason; returning to active never restores it.
7. Blocked state does not affect project/cycle completion calculations.
8. Archived issues are read-only; restore preserves status + blocked state.
9. Deleted issues are gone forever (no recovery); deletion clears project/cycle links from… (no — the *issue* is deleted; comments/notifications referencing it die with it).
10. Members create/edit/archive/restore issues; **only Owners and Admins delete issues** (PRD matrix).
11. History records every meaningful change (status, blocked, assignee, planning fields).
12. List/kanban view preference is stored per user per workspace, independently for issues and projects; list-only subviews never overwrite it.

---

## 5. Domain Operations

| Operation | Description | Requires |
|---|---|---|
| `createIssue` | New issue (defaults applied) + identifier allocation + assignment notification | member |
| `getIssue` | Detail + history | member |
| `listIssues` | Filtered/sorted/searchable list (all/mine/backlog/blocked/archived) | member |
| `updateIssue` | Title, description, status, priority, assignee, project, cycle, labels, due date, blocked | member (delete = Owner/Admin) |
| `changeStatus` | Workflow move incl. blocked-clearing on DONE; used by dropdown AND kanban drag | member |
| `toggleBlocked` | Set/clear flag + reason | member |
| `archiveIssue` / `restoreIssue` | Read-only round-trip preserving state | member |
| `deleteIssue` | Permanent; cascade comments/notifications | **Owner / Admin** |
| `setViewPreference` | Per-user per-workspace List/Kanban choice | member (stored via settings) |
| `updateLabel` / `deleteLabel` | Rename/recolor a label; delete unlinks it from all issues (issues untouched) | member |

---

## 6. Cross-Module Contracts

```
issues ──assignment/mention events──▶ notifications (same transaction)
issues ──projectId/cycleId refs──▶ projects / cycles (SetNull on delete)
issues ──labels──▶ labels (workspace-scoped, join table)
issues ──activity──▶ activity/history (own records; feeds dashboard feed)
```

| Contract | Detail |
|---|---|
| **notifications** | Assignee change → `notificationsService.notify(assignee, "ASSIGNED", issue)` inside the update transaction. Mentions → handled by comments module (issue-scoped) |
| **projects / cycles** | `projectId`/`cycleId` are plain FKs with `onDelete: SetNull` — project/cycle deletion unassigns issues automatically in the same DB transaction (PRD atomicity) |
| **workspace deletion** | Issues cascade (workspace contract) |
| **dashboard** | Dashboard aggregates read issue state (counts by status, blocked, overdue) via query services — no schema coupling |
| **search** | The tsvector column + GIN index live here; search module queries through the issue repository |

---

## 7. Trust Boundaries & Security Properties

1. All writes pass `requireWorkspaceMember`; delete additionally passes `requireRole(OWNER | ADMIN)`.
2. Assignee/project/cycle values are validated as **current workspace members/entities** server-side — the client can't assign cross-workspace.
3. Kanban drags are real status updates with real permission checks — the UI is a convenience, not a bypass (same endpoint as the dropdown).
4. Archived issues reject writes at the data layer (not just hidden in UI).
5. History is append-only (server-generated, never client-supplied).
6. Identifier allocation is transactional — no two issues can claim the same number, even under concurrency.

---

## 8. Non-Goals (MVP)

Per PRD §5.5 future enhancements: parent-child issues, dependencies, subtasks, custom issue types, recurring issues, templates, watchers, story points, rich text editing, file attachments.

---

## 9. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Priority scale | Propose Linear-style: `NO_PRIORITY · URGENT · HIGH · MEDIUM · LOW` — confirm |
| 2 | Label creation permission | Propose: any member can create labels (lightweight, workspace-scoped) — confirm |
| 3 | Identifier prefix | Propose `SHIP-###` (design sample) with a per-workspace sequence — confirm; prefix constant vs workspace-derived |
| 4 | List pagination | Propose cursor pagination for lists (issues can grow) — confirm |
