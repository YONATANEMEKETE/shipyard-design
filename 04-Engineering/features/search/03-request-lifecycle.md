# Search — Request Lifecycle

**Module:** `apps/api/src/modules/search`
**Status:** Draft v0.1 — 2026-08-12
**Relies on:** workspace lifecycle §5 (guard chain) · `02-data-model.md` · `04-api-design.md`

---

## 1. Overview

A single, read-only request shape: the user types (debounced ~300ms client-side) → one search endpoint → ranked, grouped results. No mutations, no background jobs. Guard chain: `requireSession → requireWorkspaceMember`.

---

## 2. Flow — Global search

```
1. GET /workspaces/{wsId}/search?q=&type=&status=&priority=&assigneeId=
      &projectId=&cycleId=&dueDateFrom=&dueDateTo=&sort=&order=&limit=
2. requireSession → requireWorkspaceMember
3. q preprocessing:
   ├── trim; empty → 200 { results: {} } (no dump)
   ├── length-capped (200 chars)             [400 SEARCH_INVALID_INPUT beyond]
   └── neutralize special characters (quotes, operators) — plain tsquery
4. scope: workspaceId (always — no cross-workspace leakage)
5. per selected type (all | issues | projects | cycles | members):
   ISSUES:    WHERE workspaceId + archivedAt IS NULL
              AND searchVector @@ plainto_tsquery('english', q)
              + optional filters (status/priority/assignee/project/cycle/dates)
              ORDER BY ts_rank(...) DESC, updatedAt DESC
              → short tokens (<3 chars) fall back to ILIKE on title
   PROJECTS:  same pattern on Project.searchVector (name + description),
              archived excluded
   CYCLES:    same pattern on Cycle.searchVector (name + goal), archived excluded
   MEMBERS:   name ILIKE '%q%' (trigram index) — workspace members only
6. group + cap: 20 per group (All) / 50 (type-filtered)
7. 200 → { q, total, results: {
     issues: IssueCard[], projects: ProjectCard[], cycles: CycleCard[],
     members: [{ id, name, role }] } }
```

**Search-as-you-type:** every keystroke after the debounce issues the same request — the endpoint is cheap (indexed `@@` + bounded limits) and stateless (no session search state, no server cache).

## 3. Flow — Suggestions (type-ahead overlay)

```
Same engine, tighter shape:
  GET /workspaces/{wsId}/search?q=&type=all&limit=5
  → titles only (issue title, project/cycle name, member name)
  → web renders the suggestions dropdown under the header search box;
    Enter or click issues the full grouped search
```

## 4. Edge Cases & Failure Handling

| Case | Behavior |
|---|---|
| Empty query | 200 with empty groups — never an error |
| Query > 200 chars | 400 `SEARCH_INVALID_INPUT` |
| Special characters / SQL-ish input | Neutralized at parse — no injection, no error |
| All-stopword query ("the and") | tsquery yields nothing → empty results, no error |
| Very short tokens ("ab") | ILIKE fallback per entity — still works |
| Archived entities | Excluded by query-time filter |
| Non-member hits the endpoint | 403 via membership guard |
| Search box spam (no debounce) | 60/min rate limit |
| Member renamed mid-session | Next query sees the new name (no caching) |

## 5. Dev vs Prod Differences

| Concern | Local dev | Production |
|---|---|---|
| FTS + trigram | Same Postgres (local container) | Neon — same SQL/behavior |
| Everything else | Same | Same |
