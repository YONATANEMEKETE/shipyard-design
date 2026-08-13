# Search — Domain Model

**Module:** `apps/api/src/modules/search`
**Status:** Draft v0.1 — 2026-08-12
**PRD source:** §5.10 Search · ADR-001 (Postgres FTS, no Meilisearch in MVP)

---

## 1. Overview & Scope

Search owns the **global lookup** across a workspace's entities: issues, projects, cycles, and members — search-as-you-type, filterable, ranked, grouped by entity type.

**In scope:**
- Global search within the **active workspace** (header search box)
- Entities: issues (title + description), projects (name + description), cycles (name + goal), members (name)
- Grouped results (All) or type-filtered ("Search within" dropdown)
- Filters + sorting on results
- Ranking (ts_rank + recency)

**Out of scope:**
- Advanced search operators, fuzzy matching, Meilisearch/Vector search → post-MVP (PRD future + ADR-001)
- Cross-workspace search (search is always scoped to the active workspace)
- Saved views → post-MVP (see open question 1)
- List-local search (project/cycle issue lists) → already covered by `GET /issues?q=` (issues module)

---

## 2. Domain Entities (searched, not owned)

| Entity | Searched fields | Match mechanism | Owner module |
|---|---|---|---|
| Issue | title + description | **tsvector** (existing generated column + GIN) | issues |
| Project | name + description | **tsvector** (new generated column — see data model) | projects |
| Cycle | name + goal | **tsvector** (new generated column) | cycles |
| Member | name | **ILIKE prefix** (people search; trigram index optional) | members |

**Search semantics:**
- Query terms are AND-combined; ranking by `ts_rank` (issue/project/cycle) — recency (`updatedAt`) breaks ties.
- Tokens too short for tsvector (`< 3 chars`) fall back to ILIKE on the same fields.
- Empty/whitespace query → empty result set (no "search everything" dump).
- Results are grouped by entity type when searching All; the UI renders group headers ("Issues · 4").
- Opening a result navigates to the entity (issues/projects/cycles open their pages; members open the member drawer).

---

## 3. Domain Invariants

1. Search is scoped to the active workspace — a member never sees results from other workspaces.
2. Searchable content matches what the member can see (membership = the only door; no special search permissions).
3. Only members of the workspace can search it.
4. Results respect archived state: **archived entities are excluded** from search (they are not part of active work).
5. Short-token queries still work (ILIKE fallback) — search never returns an error for a valid query.
6. Ranking is deterministic (ts_rank, then updatedAt desc) for stable UI.
7. Search does not mutate data; it reads through the owning modules' repositories.

---

## 4. Domain Operations

| Operation | Description | Requires |
|---|---|---|
| `searchWorkspace` | Query + filters across selected types, ranked + grouped | member |
| `suggest` | Search-as-you-type suggestions (same engine, tighter limit, issue/project/cycle titles + member names) | member |

---

## 5. Cross-Module Contracts

| Contract | Detail |
|---|---|
| **issues** | Reads `Issue.searchVector` (generated, DB-maintained) + GIN index; filters reuse issue query params semantics |
| **projects / cycles** | New tsvector generated columns added in the **search data model** (owned here, defined as migration SQL on those tables) |
| **members** | Member name lookup (ILIKE) against current workspace members only |
| **web** | Debounced (≈300ms) search-as-you-type; header search box opens the search overlay (UX) |

---

## 6. Trust Boundaries & Security Properties

1. `workspaceId` comes from the route + membership check — never from the query string.
2. Query strings are parsed defensively: length-capped (e.g., 200 chars), special characters neutralized for tsquery construction (no injection into SQL).
3. No cross-workspace result leakage by construction (scope applied before ranking).
4. Search is read-only — no mutation surface.

---

## 7. Non-Goals (MVP)

Per PRD §5.10 future: advanced operators (`in:`, `assignee:`, boolean syntax), fuzzy/typographical tolerance, result highlighting in snippets, saved views, global (cross-workspace) search, Meilisearch.

---

## 8. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Saved views | Post-MVP per PRD future list — confirm no MVP saved-filter persistence |
| 2 | Member search fields | Name only vs name + email — recommend name only (email is sensitive-ish in results) |
| 3 | Result limit per group | Propose 20 per group (All) / 50 (type-filtered) — confirm |
