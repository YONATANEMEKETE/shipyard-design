# Issues — Request Lifecycle

**Module:** `apps/api/src/modules/issues`
**Status:** Draft v0.1 — 2026-08-12
**Relies on:** workspace lifecycle §5 (guard chain) · `02-data-model.md` · `04-api-design.md`

---

## 1. Overview

Issue traffic is the **core workflow of the product** — creation, rapid status moves (dropdown + kanban drag), planning edits, blocking, and list/board queries. Three flows carry the interesting design: **create** (identifier allocation + assignment notification), **status change** (drag semantics + blocked auto-clear), and **delete** (the first cascade-heavy operation).

Guard chain for every endpoint: `requireSession → requireWorkspaceMember` (+ `requireRole(OWNER | ADMIN)` for delete only).

---

## 2. Flow — Create issue

```
1. POST /workspaces/{wsId}/issues
   body: { title, description?, priority?, projectId?, cycleId?, assigneeId?,
           labelIds?, dueDate? }                  [Zod; title required 1–200]
2. requireSession → requireWorkspaceMember
3. referential validation (all workspace-scoped):
   ├── projectId exists in THIS workspace     [else 400 ISSUE_INVALID_INPUT]
   ├── cycleId exists in THIS workspace
   ├── assigneeId is a member of THIS workspace   (no cross-workspace assignment)
   └── labelIds exist in THIS workspace
4. CREATE TRANSACTION (single):
   a. bump Workspace.lastIssueNumber → identifier = `SHIP-{n}`
      (unique constraint is the backstop; a P2002 retries with the next number)
   b. INSERT Issue (status = BACKLOG, priority = NO_PRIORITY or chosen,
      isBlocked = false)
   c. INSERT IssueActivity (CREATED)
   d. IF assigneeId set → notificationsService.notify(assignee, ASSIGNED, issue)
      — same transaction (PRD: assignment notification)
5. 201 → { issue } → UI shows it in the list/board
```

**Failure:** any step rolls back — no half-created issue, no orphan notification, no skipped sequence number (a rolled-back bump is fine; the number space simply has gaps — never reused).

---

## 3. Flow — Status change (dropdown OR kanban drag)

The dropdown and the drag hit the **same endpoint** — drag is a UI affordance, not a bypass.

```
1. PATCH /workspaces/{wsId}/issues/{issueId}
   body: { status }                              [Zod: enum]
2. requireSession → requireWorkspaceMember
3. issue must not be archived                    [else 409 ISSUE_ARCHIVED]
4. UPDATE TRANSACTION (single):
   a. UPDATE Issue.status = {status}
   b. IF new status == DONE AND isBlocked → UPDATE isBlocked = false,
      blockedReason = null                       (domain rule 6 — auto-clear)
   c. INSERT IssueActivity (STATUS_CHANGED { from, to })
5. 200 → { issue }

DRAG-SPECIFIC BEHAVIOR (web side, same endpoint):
- Optimistic move → on error: card returns to previous column + toast
- 409 (concurrent change / archived): card refreshes to the LATEST saved
  state + "not applied" notice (PRD)
- Same-column drag: no request at all (sort-driven order, no manual ranking)
```

**Concurrency note:** no optimistic locking on the issue row — the last-write-wins update is acceptable for status (PRD only demands the *client* recover gracefully, which the refresh-on-409 does). If conflicts ever need stricter handling, a `version` column is the documented upgrade path.

---

## 4. Flow — Blocked toggle

```
1. PATCH /workspaces/{wsId}/issues/{issueId}
   body: { isBlocked: true, blockedReason? }  |  { isBlocked: false }
2. guards as above; issue must not be archived   [409 ISSUE_ARCHIVED]
3. business rules:
   ├── set blocked on DONE issue   → 400 ISSUE_CANNOT_BLOCK (only unfinished)
   └── unblock → clear reason too
4. UPDATE isBlocked [+ blockedReason] + INSERT IssueActivity
   (BLOCKED / UNBLOCKED)
5. 200 → { issue } — status column unchanged; the kanban card shows the
   blocked indicator (never moves columns)
```

## 5. Flow — List / board queries

```
GET /workspaces/{wsId}/issues?view=all|mine|backlog|blocked|archived
    &status=&priority=&assigneeId=&projectId=&cycleId=&labelIds=
    &dueDateFrom=&dueDateTo=&blocked=&q=&sort=&order=&cursor=&limit=

1. requireSession → requireWorkspaceMember
2. compose query (repository):
   ├── scope: workspaceId (always)
   ├── view: all → non-archived; mine → assigneeId = session user;
   │         backlog → status=BACKLOG; blocked → isBlocked; archived → archivedAt
   ├── filters AND-combined (PRD §5.10)
   ├── q → to_tsquery over searchVector (GIN) — or ILIKE fallback for short tokens
   └── sort: createdAt | updatedAt | dueDate | priority | title (asc/desc)
3. cursor pagination (default limit 50, max 100)
4. 200 → { issues: [...], nextCursor }
   → web renders List or Kanban per the user's stored view preference
   (view preference fetched via settings; switching views re-runs the same
   query with the same filters — search/filters persist across toggles)
```

**Kanban column data:** one query, grouped by status server-side — the board never issues four queries.

## 6. Flow — Archive / Restore

```
ARCHIVE:
1. POST /workspaces/{wsId}/issues/{issueId}/archive
2. guards; must not already be archived          [409 ISSUE_ARCHIVED]
3. UPDATE archivedAt = now + INSERT IssueActivity (ARCHIVED)
4. 200 → { issue } → disappears from lists/boards; appears in Archived (list-only)

RESTORE:
1. POST /workspaces/{wsId}/issues/{issueId}/restore
2. guards; must be archived                      [409]
3. UPDATE archivedAt = null + INSERT IssueActivity (RESTORED)
4. 200 → { issue } → back in its PREVIOUS status column and blocked state
   (both were preserved untouched while archived — no snapshot to restore)
```

## 7. Flow — Delete issue (Owner / Admin)

```
1. DELETE /workspaces/{wsId}/issues/{issueId}
2. requireSession → requireWorkspaceMember → requireRole(OWNER | ADMIN)
   (matrix: members cannot delete — 403 ISSUE_FORBIDDEN)
3. confirm dialog (web) — no server-side name gate needed (single row)
4. DELETE Issue → comments + activities cascade; notifications cascade
   (dead references must not survive — workspace contract §5)
5. 204 → gone forever (no recovery); the identifier is NEVER reused
```

## 8. Edge Cases & Failure Handling

| Case | Behavior |
|---|---|
| Drag fails (network/server) | Card returns to previous column + toast; retry allowed |
| Concurrent status change during drag | 409 → card refreshes to latest saved state + "not applied" notice |
| Blocked issue moved to DONE | Flag + reason auto-cleared (activity records both) |
| Done issue moved back to active | Blocked state NOT restored (domain rule 6) |
| Block attempt on DONE issue | 400 `ISSUE_CANNOT_BLOCK` |
| Edit on archived issue | 409 `ISSUE_ARCHIVED` (data layer, not just UI) |
| Assignee removed from workspace | Membership check on next update; issue keeps assigneeId until changed (removal does not clear assignments) — *decision: assignments persist* |
| Project deleted with issues | Issues stay, projectId auto-cleared (SetNull, same txn) |
| Cycle deleted with issues | Issues stay, cycleId auto-cleared |
| Identifier collision (concurrent creates) | Unique constraint → retry with next number (no reuse, no skip-sharing) |
| Search token too short | Falls back to ILIKE on title (no GIN penalty for 1–2 char queries) |
| Empty kanban column | Rendered as drop target (PRD) — query groups by status, empty groups included |

## 9. Dev vs Prod Differences

| Concern | Local dev | Production |
|---|---|---|
| FTS | Same Postgres + GIN (local container) | Neon Postgres — same behavior |
| Notifications | Same in-transaction writes | Same |
| Rate limits | Same | Same |

