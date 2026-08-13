# Search — Data Model

**Module:** `apps/api/src/modules/search`
**Status:** Draft v0.1 — 2026-08-12
**Stack:** PostgreSQL full-text search (ADR-001) — **no new tables**
**PRD source:** §5.10 Search

---

## 1. Overview

Search adds **indexes only** — the module owns no tables. It reads through the owning modules' repositories and adds the missing search vectors:

| Entity | Search column | Index | Already exists? |
|---|---|---|---|
| Issue | `searchVector` (title + description) | GIN | ✅ (issues data model §2) |
| Project | `searchVector` (name + description) | GIN | ➕ added here |
| Cycle | `searchVector` (name + goal) | GIN | ➕ added here |
| Member | `name` — ILIKE prefix match | (optional `pg_trgm` GIN) | ➕ added here |

---

## 2. Migration SQL (added in the search feature's migration)

```sql
-- PROJECTS — generated vector, DB-maintained (same pattern as issues):
ALTER TABLE "Project" ADD COLUMN "searchVector" tsvector
  GENERATED ALWAYS AS (
    to_tsvector('english', coalesce(name,'') || ' ' || coalesce(description,''))
  ) STORED;

CREATE INDEX project_search_idx ON "Project" USING GIN ("searchVector");

-- CYCLES:
ALTER TABLE "Cycle" ADD COLUMN "searchVector" tsvector
  GENERATED ALWAYS AS (
    to_tsvector('english', coalesce(name,'') || ' ' || coalesce(goal,''))
  ) STORED;

CREATE INDEX cycle_search_idx ON "Cycle" USING GIN ("searchVector");

-- MEMBERS (people search — prefix-tolerant):
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX member_name_trgm_idx ON "User" USING GIN (lower(name) gin_trgm_ops);
```

**Prisma note:** these are migration-time statements (Prisma cannot express generated columns or trigram indexes) — the `searchVector` fields are declared in the Prisma schema as `Unsupported("tsvector")?` (same pattern as issues).

---

## 3. Design Rationale

- **Generated columns everywhere** — the DB maintains the vectors on every insert/update; application code cannot desync them (the core argument for the tsvector approach over app-side tokenization).
- **Consistent `english` config** — one stemmer across all entities keeps ranking comparable between groups.
- **`pg_trgm` for member names** — names are searched as prefixes/subsets (`%ma%` matches "Maya"); trigram GIN makes ILIKE fast and is the standard Postgres people-search pattern. At MVP scale it is optional (sequential scan is fine under 1k members) — the index is added now for correctness of the pattern, cost is negligible.
- **No denormalized search cache** — no Redis/Meilisearch sync, no shadow tables (ADR-001). The source tables ARE the search index.
- **Archived exclusion** happens at query time (`archivedAt IS NULL` filters), not in the index — keeps the index simple and correct for all queries.

---

## 4. Indexes & Constraints Summary

| Object | Type | Why |
|---|---|---|
| `Issue.searchVector` | GIN | Issue search (existing) |
| `Project.searchVector` | GIN | Project search (added) |
| `Cycle.searchVector` | GIN | Cycle search (added) |
| `User.lower(name)` | GIN (trigram) | Member name search (added) |

---

## 5. Data Lifecycle

| Event | SQL-level behavior |
|---|---|
| Any issue/project/cycle insert or update | Generated column updates automatically (DB) — **no app code** |
| Member name change | `User.name` update → trigram index updates automatically |
| Archive / restore | No index change — query-time `archivedAt` filter excludes/restores |
| Delete | Row removed → vector + index entries removed by the DB |
| Search query | Read-only: tsquery parse → `@@` matches → ts_rank ordering (per type) → ILIKE fallback for short tokens |

---

## 6. Sizing & Free-Tier Fit

GIN indexes add roughly 1× the indexed text size. At 10k issues + 1k projects + 500 cycles ≈ a few MB of index — inside Neon's 0.5GB free tier. The trigram index on member names is negligible. No TTL/cache concerns — the indexes are always current by construction.

---

## 7. Decisions Adopted (from domain model open questions)

| # | Question | Decision |
|---|---|---|
| 1 | Saved views | **Post-MVP** (PRD future list) — no saved-filter persistence in MVP |
| 2 | Member search fields | **Name only** (email excluded from search results) |
| 3 | Result limits | **20 per group** (All) / **50** (type-filtered) |
