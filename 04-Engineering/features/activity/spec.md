# Activity Log — Feature Spec

**Status:** Approved
**Last updated:** 2026-09-04
**Design sources:** User decision 2026-09-04 (new feature, no PRD section) · `features/dashboard/spec.md` (Recent Activity consumer) · `features/notifications/spec.md` (emission-side precedent — what counts as event-worthy)
**Technical design:** Produced alongside this spec as `data-model.md` + `api-design.md` in this folder.

---

## 1. What this feature is about

Activity Log owns the **workspace narrative**: a browsable, workspace-scoped record of the main events of every feature — workspace updates, membership changes, project/issue/comment/cycle lifecycle — with its own page that any member can read. Owning tables hold current *state* (an invitation is `DECLINED`); the log holds the *history* ("Maya declined the invite").

It is an append-only, snapshot-based record: event rows freeze the relevant names and titles at emit time, so deleted entities remain readable instead of vanishing with their source rows.

## 2. What users can do

- Open the workspace activity page and browse events newest-first.
- Filter by feature area (workspace / members / projects / issues / comments / cycles) and by actor.
- Click an event to navigate to the underlying issue, project, cycle, or member context where it still exists.
- Read events about deleted entities as frozen text (no dead links, no errors).
- See who did what and when for every main event in the workspace.

## 3. Main behaviors & actions

### 3.1 Event taxonomy (locked — full breadth)

| Area | Recorded events |
|---|---|
| Workspace | Created, updated (name/icon), archived, restored |
| Members | Invited, joined (accepted), declined, invite revoked, removed, left, role changed, ownership transferred |
| Projects | Created, renamed, status changed, owner transferred, archived, restored, deleted |
| Issues | Created, status changed, assigned, blocked set, blocked cleared, archived, restored, deleted |
| Comments | Created, deleted (edits excluded) |
| Cycles | Created, started, completed, reopened, archived, restored, deleted |

Excluded in MVP: label create/rename/delete, priority-only changes, project/cycle date edits, view-preference changes, resend-invite (noise, not narrative), notification reads/deletes, dashboard views.

### 3.2 Recording

- Events are recorded synchronously inside the source transaction — the event and its source commit together or not at all. A failed log write fails the source action (strict).
- Each row freezes actor display name, entity title, and a rendered summary sentence at emit time. Later renames and deletions never rewrite the row.
- The log starts empty at deploy — history begins on landing, no backfill of pre-existing rows.
- No retention cap in MVP; rows are small and unbounded growth is revisited with evidence.

### 3.3 Reading

- Workspace-scoped page, any member (including `MEMBER` role), newest-first cursor pagination.
- Deleted-entity events render as frozen text without navigation links; living entities link to their pages.
- Archived workspaces remain readable; the log of an archived workspace is itself frozen (no new events can arise inside it).

## 4. User flows (high level)

1. **Browse:** workspace → activity page → newest events with actor + summary; filter by area/actor; load more walks back in time.
2. **Trace:** invitation shows `DECLINED` → activity page filtered to members shows "Owner invited bob@…", then "bob@… declined the invite" — the full story behind the state.
3. **Drill down:** click "Maya moved SHIP-24 to In Progress" → issue detail; click an event for a deleted issue → frozen text, no link, no error.

## 5. Business rules

1. Every event belongs to exactly one workspace and names one actor — a member display name, or the invitee email for invitation-lifecycle events (the invitee may act without ever becoming a member).
2. Recording is strict and synchronous — no sourceless events, no silent holes.
3. Rows are immutable and snapshot-based — never updated, never rewritten by later renames or deletions.
4. Any member reads everything; there is no author-only or role-gated concept here.
5. Deleting an entity never deletes its log rows (inverse of notification cascades).
6. Deleting a workspace deletes its log; deleting a user nulls their actor link but keeps rows with the frozen name.
7. Per-entity histories (`issue_history`, invitation `status`) stay authoritative for their detail views — the log is the cross-entity narrative, not a replacement.

## 6. Out of scope (MVP)

Label/priority/date micro-changes, resend-invite noise, notification interactions, realtime push, per-user filtering preferences, export (CSV), retention jobs/TTL, cross-workspace feed, edit history on log rows, backfill of pre-feature history.

## 7. Open product questions — resolved

| # | Question | Decision |
|---|---|---|
| 1 | Taxonomy breadth | Full starter set above (§3.1); micro-changes excluded. |
| 2 | Snapshots vs live joins | Snapshots frozen at emit; rows survive source deletion. |
| 3 | Actor on user delete / non-member actors | Frozen display name kept; link nulled. Invitation-lifecycle rows use the invitee email (actor may never be a member). |
| 4 | Reads | `/w/:slug/activity`, any member, cursor newest-first, 25/100 bounds. |
| 5 | Retention | Uncapped in MVP. |
| 6 | Backfill | None — starts empty at deploy. |
| 7 | Emission failures | Strict — fail the source transaction. |
| 8 | Permissions | Any member reads all; archived readable. |
