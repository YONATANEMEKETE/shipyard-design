# Search — Data Model

**Status:** Draft for review
**Last updated:** 2026-09-04
**Sources:** `features/search/spec.md` · `features/issues/data-model.md` (F5 handoff — `tsvector` on title/description/identifier + GIN, `ILIKE` list behavior kept) · `features/cycles/data-model.md` (F7 handoff — `tsvector` on name/goal + GIN) · `features/comments/data-model.md` (F8 handoff — `tsvector` on content + GIN, comment cards in grouped results) · `features/projects/api-design.md` (F4 note — full-text on project name) · `features/members/api-design.md` (F3 note — directory search; email explicitly excluded here, D5) · `00-architecture.md` §5/§8/§9 + decision 12 (Postgres full-text, english config; Meilisearch post-MVP) · `ADR-001` (Prisma + Postgres) · `ADR-002` (shared contracts) · `Implementation Plan.md` F10
**Owner:** `apps/api` — Prisma-owned migration (raw-SQL columns, §8); reads via typed `$queryRaw` (§6.1).

> **Locked scope (2026-09-04):** five searchable groups (issues / projects / cycles / members / comments — comments added per discussion, capped lower); members name-only (email excluded, supersedes the F3 line); no `pg_trgm` (prefix `:*` + short-token `ILIKE`); 20/group, 50 type-filtered, suggestions via `limit=5`; empty query → `200` empty; max 200 chars; `updatedAt` desc tiebreak (+ `id` final); F5 `?q` untouched; english-only stemming.

---

## 1. Overview

Search owns the **global lookup**: ranked, grouped, workspace-scoped full-text over five entity types. It stores no domain rows of its own — its only schema footprint is four generated `tsvector` columns plus GIN indexes on tables owned elsewhere. Members (small per-workspace sets on a Better Auth-owned table) are served by scoped `ILIKE`, not by touching the `user` table definition.

Schema footprint (all additive, all in one F10 migration):

| Change | Purpose | Formalized by |
|---|---|---|
| `issue.search_tsv` | Weighted `title(A) + description(B)` lexemes | **F10 (this milestone)** |
| `project.search_tsv` | Weighted `name(A) + description(B)` lexemes | **F10** |
| `cycle.search_tsv` | Weighted `name(A) + goal(B)` lexemes | **F10** |
| `comment.search_tsv` | Weighted `content(B)` lexemes | **F10** |
| GIN index per column | Ranked-match hot path | **F10** |

No new tables, no new enums (the `type` filter is a query param over a fixed five-way union, §4). Notifications, activity events, invitations, labels, view trails, and dashboard panels are never searched (each excluded in its own data-model §7).

---

## 2. Core schema (migration-owned, Prisma-unmapped)

Prisma's schema language cannot express generated `tsvector` columns, so F10 follows the established raw-SQL-append pattern (`workspace_single_owner`, `cycle_no_overlap`): the migration adds the columns + indexes as SQL, and the Prisma schema stays untouched. Reads go through typed `$queryRaw` (§6.1) — there is nothing for Prisma Client to map, and nothing to drift.

### 2.1 Generated columns (english config, weighted)

```sql
-- Issues: title outranks description (D3)
ALTER TABLE issue ADD COLUMN search_tsv tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'B')
  ) STORED;
CREATE INDEX issue_search_gin ON issue USING GIN (search_tsv);

-- Projects: name outranks description
ALTER TABLE project ADD COLUMN search_tsv tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'B')
  ) STORED;
CREATE INDEX project_search_gin ON project USING GIN (search_tsv);

-- Cycles: name outranks goal
ALTER TABLE cycle ADD COLUMN search_tsv tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(goal, '')), 'B')
  ) STORED;
CREATE INDEX cycle_search_gin ON cycle USING GIN (search_tsv);

-- Comments: body only (single weight — no title field exists)
ALTER TABLE comment ADD COLUMN search_tsv tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(content, '')), 'B')
  ) STORED;
CREATE INDEX comment_search_gin ON comment USING GIN (search_tsv);
```

Generated columns backfill automatically on `ALTER TABLE` (existing rows computed at migration time) and stay fresh on every write — no triggers, no backfill job, no application code. `coalesce` keeps nullable bodies (`description`, `goal`) from nulling the whole vector.

### 2.2 Members leg — scoped `ILIKE`, no schema change (D5)

```sql
-- No migration change. Served at read time:
-- SELECT u.* FROM workspace_member m JOIN "user" u ON u.id = m.user_id
--  WHERE m.workspace_id = ? AND u.name ILIKE '%<trimmed>%'
```

Rationale: per-workspace member sets are tiny (tens, not thousands), `user` is Better Auth-owned (F1-generated — hand-editing around it risks upgrade drift), and name-only matching (spec Q2) needs no lexeme machinery. If directory sizes ever justify it, the exit ramp is a `tsvector` on an F-owned `member_search` view — not in MVP.

### 2.3 What is deliberately not indexed

`SHIP-###` identifiers keep their exact-match short-circuit (F5 behavior — numerals stem poorly, so identifier lookup never touches `tsvector`, §6.2). Label names, invitation emails, notification text, activity summaries, and view trails have no columns and no legs.

---

## 3. Key decisions & alternatives

### D1 — Generated `STORED` columns, not triggers or application-maintained vectors

**Decision:** `GENERATED ALWAYS ... STORED` — the database owns freshness transactionally (a title edit and its lexemes commit atomically, always). *Rejected:* (a) update triggers — second mechanism doing the same job with its own failure modes; (b) application-computed vectors on write — every writer (create/edit/rename paths across four modules) would need identical lexing code, and any missed path silently de-indexes rows.

### D2 — Raw-SQL migration append, Prisma schema untouched

**Decision:** same pattern as `cycle_no_overlap` / `invitation_single_pending`: SQL lives in the migration folder, Prisma schema gains nothing, reads use typed `$queryRaw` returning owning card shapes (§6.1). *Rejected:* `Unsupported("tsvector")` mapped fields — mapping buys nothing (Prisma still can't query/rank them) while advertising a column the application must never write (generated columns reject writes).

### D3 — Title/name weight `A`, body weight `B`

**Decision:** `setweight` at generation (not query) time, so "checkout" in a title outranks "checkout" buried in a description with zero per-query cost. Comments get uniform `B` (no title exists to promote). Weights are generation-time constants — re-tuning means one migration rewriting four expressions, documented here so it isn't done casually.

### D4 — Prefix `:*` per term + short-token `ILIKE` leg, no `pg_trgm` (locked)

**Decision:** query `plainto_tsquery`-shaped terms each suffixed `:*` (as-you-type "che" hits "checkout"; multi-term AND semantics preserved — spec §3.1), OR-ed per row with a whole-query `ILIKE '%trimmed%'` containment leg that guarantees spec rule 4 (short/partial tokens always match). Tokens under 2 chars skip `tsquery` entirely (the stemmer discards most of them) and run `ILIKE`-only. *Rejected:* `pg_trgm` + trigram GINs — buys typo-tolerance the spec excludes (§6), at the price of a second index family and another extension install next to `btree_gist`. If partial-matching ever goes slow (`%..%` can't use B-tree), trigrams are the named exit ramp — with evidence, not preemptively.

### D5 — Members name-only `ILIKE`, email excluded (locked, supersedes F3)

**Decision:** the member leg matches `user.name` only. Email searchability would let anyone with membership enumerate addresses by prefix-guessing — spec Q2's sensitivity call stands, and the F3 api-design line ("full-text on `user.name`/`email`") is superseded in writing here. Directory client-side filtering (F3) stays as-is for the members page; this leg serves grouped search only.

### D6 — Archived excluded on every leg (spec rule 3)

**Decision:** `issue/project/cycle` legs carry `archivedAt IS NULL`; the comment leg joins its issue and requires the issue non-archived (comments have no flag of their own — showing discussion from archived work would leak frozen content into active search); the member leg needs no predicate (no archived state; departed members leave via membership deletion). Archived-workspace reads stay allowed (workspace freeze ≠ entity archive — same tolerance as dashboard).

### D7 — Deterministic order: `ts_rank DESC, updatedAt DESC, id ASC` (locked)

**Decision:** rank first (spec: relevance), recency second (`updatedAt` uniform across all five tables), `id` last so equal-rank/equal-time rows still order byte-identically run to run (rule 5 testable — same query twice ⇒ byte-equal bodies). No cross-group ranking: each group orders internally; groups are independent result sets, not one merged list.

### D8 — Grouped endpoint with per-group bounds, no merged relevance (locked)

**Decision:** `GET .../search` returns five bounded arrays (`~20` each, `50` when `?type=` filters to one, client sends `limit=5` for suggestions). *Rejected:* one merged top-N across types — cross-type rank comparability is meaningless (issue-title-A-weight vs member-ILIKE have no common scale) and the UI renders groups anyway ("Issues · 4"). Bounds from spec Q3 verbatim.

---

## 4. Shared contracts (`packages/shared`)

Added in F10, consumed by `api` and `web` (ADR-002). Entity cards re-exported from owning modules — never redefined — except the comment hit, which needs its issue context for grouped display.

```ts
// query contract owned by the search module
export const searchTypeSchema = z.enum(["issues", "projects", "cycles", "members", "comments"]);

export const searchQuerySchema = z.object({
  q: z.string().trim().min(1).max(200), // empty/whitespace handled as 200-empty, never 400 (§5)
  type: searchTypeSchema.optional(),    // omitted ⇒ all groups ("search within" dropdown)
  limit: z.coerce.number().int().min(1).max(50).optional(), // per-group bound; default 20 (50 when type set — api-design)
});

// comment hits need their issue context for grouped display (permalink target)
export const searchCommentHitSchema = commentCardSchema.extend({
  issueId: z.string(),        // duplicates card.issueId — explicit for hit rendering
  issueIdentifier: z.string(), // SHIP-### verbatim
  issueTitle: z.string(),
});

// grouped response — five bounded arrays, independent orders (D8)
export const searchResultsSchema = z.object({
  q: z.string(), // echo of the trimmed query
  issues: z.array(issueCardSchema),
  projects: z.array(projectCardSchema),
  cycles: z.array(cycleCardSchema),
  members: z.array(workspaceMemberCardSchema),
  comments: z.array(searchCommentHitSchema),
});
```

When `?type=` is set, only that group's array populates (others `[]`) with the bound raised to 50. Suggestions use the same schema with `limit=5` — no separate suggestion shape (§3.3: same search, tighter limit).

---

## 5. Integrity invariants → spec rule mapping

| Spec rule | Enforcement point |
|---|---|
| 1 — workspace-scoped, never leaks | `workspaceId = ?` (or membership join) on every leg; cross-workspace test per leg (api-design) |
| 2 — sees exactly what's visible elsewhere | Same non-archived predicates as list pages + membership-scoped member leg; no privileged rows (no Owner-only content exists in the five types) |
| 3 — archived excluded | D6 predicates on all four content legs (+ issue-join gate for comments) |
| 4 — short/partial matches, never an error | D4 dual leg (`:*` prefix OR `ILIKE` containment; <2 chars `ILIKE`-only); blank query ⇒ `200` empty groups |
| 5 — deterministic ranking | D7 three-key order per group; byte-equal-repeat test |
| 6 — read-only | No write path exists: generated columns maintain themselves (D1), no `POST` route, no mutation helper |

---

## 6. Lifecycle semantics (reads only — plus the one write-adjacent fact)

Search has no lifecycle: no rows, no states, no transitions. This section fixes the two read shapes and records the single freshness argument.

### 6.1 Ranked leg (issues / projects / cycles / comments)

Per entity, one `$queryRaw` statement shaped identically:

```sql
-- shape (issue leg shown; others substitute columns/cards):
SELECT <card columns>, ts_rank(search_tsv, q) AS rank
FROM issue, plainto_tsquery('english', :terms) q   -- terms = trimmed input, each suffixed :*
WHERE workspace_id = :ws
  AND archived_at IS NULL
  AND (search_tsv @@ q OR coalesce(title,'') || ' ' || coalesce(description,'') ILIKE :contains)
ORDER BY rank DESC, updated_at DESC, id ASC
LIMIT :bound;
```

- `:contains` is `%trimmed%` (whole-query containment — the rule-4 guarantee).
- Short queries (<2 chars after trim): skip the `@@` arm (still syntactically valid — `tsquery` of nothing matches nothing — but the `ILIKE` arm carries the leg; documented so the next reader doesn't "optimize" it away).
- Exact-identifier short-circuit runs **before** the legs (§6.2): on `/^SHIP-(\d+)$/i`, the issues array is the workspace-scoped exact row (subject to the archived flag), other groups run the normal legs — same composition as F5's `q` behavior, now grouped.

### 6.2 `SHIP-###` handling + F5 `?q` preservation

Identifier input (`/SHIP-(\d+)/i`, case-insensitive, trimmed) resolves the exact non-archived issue in-workspace first; a hit leads the issues group, remaining issue slots fill by rank. The F5 issues-list `?q` (`ILIKE` + exact, `api-design.md` §5.1) is byte-for-byte unchanged — grouped search is additive (locked §5-Q7).

### 6.3 Members leg (scoped `ILIKE`, D5)

```sql
SELECT <member card columns> FROM workspace_member m JOIN "user" u ON u.id = m.user_id
 WHERE m.workspace_id = :ws AND u.name ILIKE :contains
 ORDER BY u.name ASC, m.created_at ASC, m.id ASC
 LIMIT :bound;
```

Name-ascending (no rank exists to sort by) with id tiebreak — deterministic per rule 5. Email never a predicate, never returned beyond the card's existing fields (member cards already carry email to members — search adds no new exposure).

### 6.4 Freshness (the write-adjacent fact)

Generation is transactional (D1): a committed title edit is a committed lexeme set — search-after-write reads-your-write within the same Postgres, no lag, no reindex job, no stale window to test. Migration-time backfill is automatic (stored generation computes over existing rows during `ALTER TABLE`).

---

## 7. Forward handoffs

| Consumer | Contract F10 provides | Landed |
|---|---|---|
| **Issues (F5)** | Nothing changes — `?q` behavior preserved byte-for-byte (§6.2) | — (untouched by design) |
| **Projects / Cycles / Comments** | Nothing changes — list reads keep exact-match params; `search_tsv` is write-invisible to them | — (untouched by design) |
| **Dashboard (F9)** | Nothing — hub panels are not searchable surfaces | — (intentionally none) |
| **Meilisearch (post-MVP)** | Exit ramp documented (arch decision 12): dual-write or CDC to an external index only when English-stemming or scale complaints arrive with evidence — not a second query path in MVP | — (future) |

---

## 8. Migration workflow

Hand-modeled migration with raw-SQL appends (Prisma cannot express generated `tsvector` — same dispensation as `cycle_no_overlap`):

```bash
# 1 — empty Prisma migration scaffold (no schema.prisma change — D2):
pnpm --filter @shipyard/api db:migrate -- --name add_search_vectors --create-only
# 2 — append the four ALTER TABLE + four CREATE INDEX statements (§2.1) to the scaffold SQL
# 3 — run
pnpm --filter @shipyard/api db:migrate
pnpm --filter @shipyard/api db:generate  # no-op for schema, keeps client in sync
```

- The migration produces: 4 generated columns + 4 GIN indexes. Zero Prisma model changes, zero backfill jobs (stored generation computes over existing rows inline — large tables will hold an `ACCESS EXCLUSIVE` lock during the rewrite; acceptable in MVP, noted for future online-migration care).
- The F1 Testcontainers harness applies migrations automatically each test run — ranked-query tests run against real GIN behavior, never mocks.

**Post-migration verification (manual, once):**

```sql
-- columns exist, generated, populated (no NULL vectors on rows with content)
SELECT count(*) FILTER (WHERE search_tsv IS NULL) FROM issue;
SELECT count(*) FILTER (WHERE search_tsv IS NULL) FROM project;
SELECT count(*) FILTER (WHERE search_tsv IS NULL) FROM cycle;
SELECT count(*) FILTER (WHERE search_tsv IS NULL) FROM comment;
-- GIN indexes exist
SELECT indexname FROM pg_indexes WHERE indexname IN
  ('issue_search_gin','project_search_gin','cycle_search_gin','comment_search_gin');
-- ranking sanity: a title term outranks a body-only term (D3 weights)
SELECT id, title, ts_rank(search_tsv, plainto_tsquery('english','checkout'))
  FROM issue WHERE workspace_id = '<seed>' AND search_tsv @@ plainto_tsquery('english','checkout')
  ORDER BY 3 DESC LIMIT 5;
```

---

## 9. What we intentionally do NOT model

| Deferred | Why |
|---|---|
| `pg_trgm` / trigram indexes | Rejected in D4 — typo-tolerance excluded by spec §6; `: *`+`ILIKE` covers as-you-type. Named exit ramp with evidence. |
| Per-leg or merged write paths | Rejected — read-only feature (rule 6); D1 generation is the only "write" and it is declarative. |
| Mapped Prisma fields for `search_tsv` | Rejected in D2 — unmappable-for-query columns invite hand-written writes into generated columns (which Postgres rejects at runtime). |
| Email/AUTH-table vectors | Rejected in D5 — Better Auth-owned table + sensitivity; member leg stays `ILIKE`. |
| Label / notification / activity / view-trail legs | No handoff promised them (§1 table); labels are browsed via the manager, not searched. |
| Advanced operators, boolean syntax | Spec §6 out of scope. |
| Fuzzy/typo tolerance | Spec §6 out of scope (see `pg_trgm` row). |
| Highlighting / snippets | Spec §6 out of scope — cards render full titles; comment hits show content head client-side. |
| Saved views / saved filters | Spec Q1 resolved: none — `type`+query live in the URL, not storage. |
| Cross-workspace search | Spec §6 out of scope — rule 1 is absolute. |
| External engines (Meilisearch) | Post-MVP per arch decision 12 (§7). |

---

## 10. Open product questions — resolved at data layer

| Spec §7 | Decision |
|---|---|
| 1 — saved views | **None.** No persistence for queries/filters; URL carries state (§9). |
| 2 — member search fields | **Name only.** Email excluded (D5); supersedes the F3 api-design line in writing. |
| 3 — result limits | **Locked:** ~20 per group, 50 type-filtered, suggestions via `limit=5` (D8, §4). |

---

## 11. References

- Shipyard: `features/search/spec.md`, `features/issues/data-model.md` + `api-design.md` §5.1 (F5 `q` contract, identifier short-circuit), `features/cycles/data-model.md` (F7 handoff), `features/comments/data-model.md` + `api-design.md` #2 (F8 handoff, permalink convention), `features/projects/api-design.md` §11 (F4 note), `features/members/api-design.md` §11 (F3 note, superseded on email), `features/auth/data-model.md` (Better Auth `user` ownership), `00-architecture.md` §5/§8/§9 + decision 12, `ADR-001`, `ADR-002`, `Implementation Plan.md` F10
- PostgreSQL full-text search: `https://www.postgresql.org/docs/current/textsearch.html`
- PostgreSQL generated columns: `https://www.postgresql.org/docs/current/ddl-generated-columns.html`
- PostgreSQL GIN + `tsvector`: `https://www.postgresql.org/docs/current/textsearch-indexes.html`
- Prisma raw SQL reads: `https://www.prisma.io/docs/orm/prisma-client/queries/raw-database-access`

---

*Next artifact: `api-design.md` — grouped endpoint inventory (`GET` search with `q`/`type`/`limit`, suggestions as `limit=5`), workspace-context guard chain (any member, readable-when-archived), error codes, per-leg sequences (prefix, short-token, identifier, archived-exclusion), and the F5-`?q`-preservation note.*
