# Search — Feature Spec

**Status:** Approved
**Last updated:** 2026-08-22
**Design sources:** PRD §5.10 · UX (header search)
**Technical design:** Excluded by design — produced during this feature's implementation step, driven by this behavioral spec.

---

## 1. What this feature is about

Search owns the **global lookup** across a workspace's entities: issues, projects, cycles, and members — search-as-you-type, filterable, ranked, grouped by entity type. Search is always scoped to the **active workspace**; there is no cross-workspace search.

## 2. What users can do

- Search the active workspace from the header search box.
- Find issues (title + description), projects (name + description), cycles (name + goal), and members (name).
- Search as they type (debounced, results update while typing).
- See grouped results ("Issues · 4", "Projects · 1", …) or narrow to one entity type ("Search within" dropdown).
- Filter and sort results; see the most relevant matches first.
- Open a result and navigate directly to the entity (issues/projects/cycles open their pages; members open the member drawer).

## 3. Main behaviors & actions

### 3.1 Query semantics
- Query terms are combined (all terms must match; relevance-ranked).
- Relevance first (best matches), recency breaks ties — deterministic ordering for a stable UI.
- Short tokens still find matches (search never errors on a valid short/partial query).
- Empty or whitespace-only queries return nothing (no "search everything" dump).

### 3.2 Scope & visibility
- Only the active workspace is searched — a member never sees results from other workspaces.
- Searchable content matches what the member can see (membership is the only door).
- **Archived entities are excluded** from results (they are not part of active work).

### 3.3 Suggestions
- Search-as-you-type suggestions use the same search, with a tighter result limit (issue/project/cycle titles + member names).

## 4. User flows (high level)

1. **Search:** header box → type → grouped results as you type → pick an entity group or within-filter → click result → entity opens.
2. **No results:** valid query, nothing found → designed no-result state (not an error).
3. **Member result:** open the member drawer from a search hit.

## 5. Business rules

1. Search is scoped to the active workspace — no cross-workspace leakage, ever.
2. Results respect visibility: a member sees exactly what they can see elsewhere.
3. Archived entities are excluded from results.
4. Short/partial tokens still return matches — never an error for a valid query.
5. Ranking and ordering are deterministic.
6. Search is read-only.

## 6. Out of scope (MVP)

Advanced operators (`in:`, `assignee:`, boolean syntax), fuzzy/typographical tolerance, result highlighting/snippets, saved views, global (cross-workspace) search, external search engines.

## 7. Open product questions

| # | Question | Notes |
|---|---|---|
| 1 | Saved views | Post-MVP — confirm no saved-filter persistence |
| 2 | Member search fields | Name only recommended (email is sensitive) — confirm |
| 3 | Result limits | ~20 per group / 50 type-filtered — confirm |
