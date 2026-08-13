# Projects — Request Lifecycle

**Module:** `apps/api/src/modules/projects`
**Status:** Draft v0.1 — 2026-08-12
**Relies on:** workspace lifecycle §5 (guard chain) · `02-data-model.md` · `04-api-design.md`

---

## 1. Overview

Project traffic mirrors issues (list/board/detail/edit) plus three project-specific flows: **create** (Owner/Admin + name uniqueness), **ownership transfer** (the TOCTOU-safe swap), and **delete** (SetNull unassignment in one DB statement). Guard chain: `requireSession → requireWorkspaceMember`, with `requireRole(OWNER | ADMIN)` on create/transfer/archive/restore/delete.

---

## 2. Flow — Create project

```
1. POST /workspaces/{wsId}/projects
   body: { name, description?, startDate?, targetDate? }   [Zod]
2. requireSession → requireWorkspaceMember → requireRole(OWNER | ADMIN)
   (matrix: members cannot create projects)
3. name normalization: trimmed; conflict check on nameNormalized
   → 409 PROJECT_NAME_CONFLICT (archived projects reserve names too)
4. TRANSACTION:
   a. INSERT Project (status PLANNED, ownerId = creator, name + nameNormalized)
   b. INSERT ProjectActivity (CREATED)
5. 201 → { project } → web redirects to project details
```

## 3. Flow — Update project (incl. free status switch)

```
1. PATCH /workspaces/{wsId}/projects/{projectId}
   body: { name?, description?, startDate?, targetDate?, status? }  [Zod]
2. guards: member (matrix: edit = all roles)
3. must not be archived                        [409 PROJECT_ARCHIVED]
4. if name changed → normalized conflict check [409 PROJECT_NAME_CONFLICT]
5. status: any of PLANNED/ACTIVE/COMPLETED — free switch, NO confirmation,
   Archived never accepted (400 PROJECT_INVALID_INPUT)
6. TRANSACTION: UPDATE + INSERT ProjectActivity (STATUS_CHANGED {from,to}
   when status changed)
7. 200 → { project }
```

## 4. Flow — Transfer project ownership

```
1. POST /workspaces/{wsId}/projects/{projectId}/transfer-ownership
   body: { userId }                             [Zod: cuid]
2. guards: requireRole(OWNER | ADMIN)
3. project must NOT be archived                 [409 PROJECT_ARCHIVED]
   (ownership controls hidden while archived — UX decisions doc)
4. TRANSACTION:
   a. re-verify target is a CURRENT workspace member (inside txn — no TOCTOU;
      same pattern as workspace ownership transfer)
      → not a member → 404 PROJECT_NOT_FOUND
   b. UPDATE Project.ownerId = target
   c. INSERT ProjectActivity (OWNER_CHANGED { from, to })
   — recipient's workspace role and permissions UNCHANGED (PRD: ownership
     grants no permissions)
5. 200 → { project } — new owner badge everywhere on next read
```

## 5. Flow — List / board queries

```
GET /workspaces/{wsId}/projects?status=&ownerId=&startDateFrom=&startDateTo=
    &targetDateFrom=&targetDateTo=&sort=&order=

1. requireSession → requireWorkspaceMember
2. scope workspaceId; filters AND-combined; archived EXCLUDED here
   (archived projects live in their own list — never in these results, PRD)
3. progress computed per project in the same query:
   COUNT(issues) / COUNT(status = DONE) — one grouped query, no N+1
4. 200 → { projects: [...], } — web renders List (Active/Planned/Completed
   sections) or Kanban (columns) per the user's stored preference; search,
   filters and sort persist across the toggle (PRD)
```

**Kanban data:** same response, grouped server-side into the three columns — one request, no four-query board.

## 6. Flow — Archive / Restore

```
ARCHIVE:
1. POST /workspaces/{wsId}/projects/{projectId}/archive
2. guards: requireRole(OWNER | ADMIN); must NOT be archived [409]
3. UPDATE archivedAt = now (operational status column UNTOUCHED — it is the
   restore target) + INSERT ProjectActivity (ARCHIVED)
4. 200 → read-only; name stays reserved; gone from boards/lists

RESTORE:
1. POST /workspaces/{wsId}/projects/{projectId}/restore
2. guards as above; must be archived                [409]
3. UPDATE archivedAt = null + INSERT ProjectActivity (RESTORED)
4. 200 → back to its STORED Planned/Active/Completed status (PRD)
```

## 7. Flow — Delete project (with atomic unassignment)

```
1. DELETE /workspaces/{wsId}/projects/{projectId}
2. guards: requireRole(OWNER | ADMIN)
3. web confirm dialog: "deletion is permanent — N issues will be unassigned,
   not deleted" (PRD screen inventory; the count comes from the detail read)
4. DELETE Project
   → Issue.projectId auto-SetNull in the SAME DB transaction (issues data
     model §3 — atomic by definition, no app orchestration)
   → ProjectActivity rows cascade; the project's name is released (row gone)
5. 204 → response confirms how many issues were unassigned
   (count read before delete, returned in a header or body note)
```

**Why SetNull is correct here:** the PRD demands issues survive with cleared assignments — SetNull is exactly that, and Postgres guarantees it happens in the same atomic statement as the delete.

## 8. Flow — Auto-transfer on member removal (internal)

```
Called by the members service INSIDE its remove/leave transaction:
  projectsService.transferOwnedProjects(userId → workspaceOwnerId)
    → UPDATE Project SET ownerId = workspaceOwnerId
       WHERE workspaceId = $1 AND ownerId = $userId
    → includes ARCHIVED projects (system-level exception — an archived
      project never retains a former member as owner, UX decisions doc)
    → + ProjectActivity (OWNER_CHANGED) per affected project
Failure of this call rolls back the whole member removal (members lifecycle §5).
```

## 9. Edge Cases & Failure Handling

| Case | Behavior |
|---|---|
| Name conflict (create or rename) | 409 `PROJECT_NAME_CONFLICT` — including archived projects reserving names |
| Member attempts create/transfer/delete | 403 `PROJECT_FORBIDDEN` (matrix) |
| Write on archived project | 409 `PROJECT_ARCHIVED` (data layer) |
| Status control receives Archived | 400 `PROJECT_INVALID_INPUT` — never a selectable status |
| Transfer target left mid-flight | Re-verified inside the transaction → 404, nothing changes |
| Drag fails / concurrent change | Same recovery contract as issues: revert + error, or refresh + notice |
| Delete while issues exist | Issues survive, projectId cleared — same DB statement |
| Owner removed from workspace | Auto-transfer (flow 8) — project never orphaned |
| Project with zero issues | Valid; progress shows 0% (or "no issues" state per empty-states doc) |

## 10. Dev vs Prod Differences

| Concern | Local dev | Production |
|---|---|---|
| Everything | Same schema/behavior (Postgres both sides) | Same |

