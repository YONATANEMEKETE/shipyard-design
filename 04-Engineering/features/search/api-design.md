# Search — API Design

**Status:** Draft for review
**Last updated:** 2026-09-04
**Sources:** `features/search/spec.md` · `features/search/data-model.md` (locked — generated vectors + GIN, D1–D8) · `features/workspace/api-design.md` (F2 precedent — `:slug` context, read-when-archived, error envelope, leak-free 404s) · `features/issues/api-design.md` §5.1 (F5 `?q` contract — preserved, never altered) · `features/comments/api-design.md` #2 (permalink convention for comment hits) · `features/members/api-design.md` (directory as member source) · `features/auth/api-design.md` (F1 — Better Auth session) · `00-architecture.md` §5–§8 · `ADR-001`–`ADR-003` · `Implementation Plan.md` F10
**Owner:** `apps/api` — hand-written Shipyard code through the canonical pipeline (`route → validation → permission check → controller → service → repository → Prisma/$queryRaw`).

---

## 1. Base path & conventions

| Concern | Choice |
|---|---|
| Base path | `/api/v1/workspaces/:slug/search` — workspace-scoped collection (rule 1 is absolute — no global variant, unlike notifications). Single `GET` route; suggestions are `limit=5` on the same route (§2). |
| Next.js proxy | Browser never hits the API directly (ADR-003); `apps/web` forwards `/api/v1/*` → `http://api:4000/api/v1/*`, cookies forwarded. |
| Auth transport | HttpOnly Better Auth session cookie read by `requireSession` (F1). |
| Validation | Zod query schemas from `packages/shared` at the route boundary (`q` trim 1–200, `type`, `limit`). Blank-after-trim `q` ⇒ `200` empty groups (never 400 — spec rule 4). |
| Envelope | Success: `searchResultsSchema` directly (with `q` echo). Failure: `{ "error": { "code", "message", "details"? } }` via the global error handler. |
| Workspace context | Reuses F2 `resolveWorkspaceContext(:slug)` with `rejectArchived: false` — archived workspaces stay searchable over their non-archived entities. |
| Read-only | No body, no mutations, no per-row routes — the only input is the query string (rule 6). |

---

## 2. Endpoint inventory

One endpoint covers every behavior in `spec.md` §2–§5. Suggestions, "search within," and deep-linking are **not** endpoints (§5.2). No extras.

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 1 | `GET` | `/api/v1/workspaces/:slug/search` | `requireSession` → member (any role) | §2/§3 Grouped ranked search. Query: `?q=` (required param, 1–200 after trim) · `?type=issues\|projects\|cycles\|members\|comments` ("search within" — only that group populates, bound 50) · `?limit=1..50` (per-group bound; default 20, suggestions pass 5). Five bounded arrays, independent orders. |

> **Why one route for search + suggestions + within-filter:** §3.3 fixes suggestions as "the same search, tighter limit" — a second path would fork ranking logic and drift. The `type` filter and `limit` param express all three surfaces with zero duplication (same reasoning as F4's one-list-serves-List+Kanban).
>
> **Why no detail routes:** hits carry full cards (ids included) — navigation targets the owning pages/drawer, which own their detail contracts.

---

## 3. Context resolution

Reuse F2's resolver verbatim: `resolveWorkspaceContext(:slug)` → `{ workspaceId, ... }`, generic `404 WORKSPACE_NOT_FOUND` on miss/non-membership. No role check (visibility is membership-only — rule 2). Every leg then scopes to `workspaceId` (members leg via the membership join); a foreign-workspace id smuggled anywhere matches zero rows — there is no id parameter to leak through.

---

## 4. Guard chain (canonical, read-only)

```text
requireSession                     ← F1: valid Better Auth session else 401
  │
resolveWorkspaceContext(slug)      ← F2, rejectArchived: FALSE → 404 generic / 200 when archived
  │
controller → service               ← Zod query validation → five bounded legs (§8.1) → grouped assembly
```

No `requireWorkspaceRole`, no confirmations, no mutations — the chain's only job after scope is bounding the query (length cap, per-group limits).

---

## 5. Request/response contracts

### 5.1 Grouped search

| Endpoint | Query | Success response |
|---|---|---|
| #1 search | `?q=<text>` (required key; trim 1–200) · `?type=` (optional group filter) · `?limit=1..50` (per-group; default `20`, or `50` when `type` is set — spec Q3) | `200` + `searchResultsSchema` `{ q (echo), issues[≤bound], projects[≤bound], cycles[≤bound], members[≤bound], comments[≤bound] }` |

Validation & behavior details:

- **Missing `q` key** ⇒ `400 VALIDATION_ERROR` (the param is required). **Present-but-blank** (`?q=%20%20`) ⇒ `200` with all groups `[]` and `q: ""` echo — never an error, never a dump (spec §3.1).
- **`q` > 200 chars after trim** ⇒ `400 VALIDATION_ERROR` (F5 precedent — aborts table-scan abuse at the edge).
- **`<2` chars** ⇒ legs run `ILIKE`-only (stemmer-skip, data-model D4) — still `200` with real matches (rule 4), just unranked-by-lexeme (recency order within the bound).
- **`?type=`** unknown value ⇒ `400 VALIDATION_ERROR`; known value ⇒ only that array populates (others `[]`), bound defaults to 50 unless an explicit smaller `limit` is passed.
- **`?limit=`** out of range ⇒ `400 VALIDATION_ERROR`. It bounds *each* group, not the total — groups are independent result sets (D8).
- **Identifier input** (`/^SHIP-(\d+)$/i`): issues array leads with the workspace-scoped exact non-archived row when it exists; remaining slots fill by rank; other groups run normally (§8.3).
- **Archived entities never appear** in any array (D6) — including comments whose issue is archived and rows in archived workspaces' *archived* subsets. No `archived=true` variant exists here (unlike list pages — search is active-work-only by spec rule 3).
- **Member hits** return existing member cards (name-ordered, id-tiebreak) — email is never a predicate and no new fields are exposed (§8.4).
- **Comment hits** return `searchCommentHitSchema` (card + `issueIdentifier`/`issueTitle`) linking to the issue permalink `#comment-<id>` (F8 #2 convention).

### 5.2 Non-endpoints (explicitly not routes)

| Concern | Served by | Notes |
|---|---|---|
| Suggestions | #1 with `limit=5` (client-debounced ~250ms) | Titles + names surface first via A-weights; no separate shape. |
| "Search within" | #1 with `?type=` | Dropdown maps 1:1 to `searchTypeSchema`. |
| Entity detail | Owning pages/drawer (issues/projects/cycles/member drawer) | Hit cards carry the ids/slugs navigation needs. |
| Issues-list `?q` | Unchanged F5 contract (§8.5) | Additive feature — zero drift on list filtering. |

---

## 6. Archived / state matrices

| Situation | #1 search |
|---|---|
| Workspace `ARCHIVED` | ✅ allowed — searches non-archived entities normally |
| Entity archived (any of the four content types) | Excluded from its group (rule 3) |
| Comment on archived issue | Excluded (issue-join gate, D6) |
| Member departed | Absent (membership join) |
| Empty workspace / no matches | ✅ `200` with empty arrays + designed no-result state (spec §4.2 — not an error) |

---

## 7. Error codes (Search module)

| Code | HTTP | When | Notes |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | Missing `q` key, `q` > 200 chars, bad `type`/`limit` | `details` lists field paths. Blank-but-present `q` is NOT an error (§5.1). |
| `WORKSPACE_NOT_FOUND` | 404 | Unknown `:slug` or caller not a member — identical | No existence leak |
| `UNAUTHENTICATED` | 401 | Missing/expired session cookie | F1 |
| `RATE_LIMITED` | 429 | Keystroke-scale abuse guard (wiring F12; debounce is the first defense) | `Retry-After` |

No `FORBIDDEN_ROLE` (no roles), no `*_ARCHIVED` 409s (exclusion, not rejection), no per-group codes (a failing leg is a `500` — partial groups would mislead like dashboard partials).

---

## 8. Sequences

### 8.1 Grouped search (the only sequence that matters)

```text
Member types "check" (debounced) → GET .../search?q=check&limit=20
→ requireSession ✓ → resolveWorkspaceContext ✓ (any member) → Zod (trim → "check", ≤200 ✓)
→ service fan-out (parallel, each bounded — five legs, §6.1–§6.3 of data-model):
     ├─ issues:   tsquery(check:*) @@ search_tsv OR containment → rank, updatedAt, id → ≤20 cards
     ├─ projects: same shape → ≤20    ├─ cycles: same shape → ≤20
     ├─ members:  name ILIKE %check% → name order → ≤20
     └─ comments: same FTS shape + issue-join gate → ≤10 hits with issue context
→ 200 { q: "check", issues: [...], projects: [...], cycles: [...], members: [...], comments: [...] }
→ any leg throws → 500 + retry (no partial groups, §7)
```

### 8.2 Suggestions (keystroke path)

```text
Header box, per debounced keystroke → GET .../search?q=che&limit=5
→ same legs, bound 5 → compact dropdown (titles + names lead via A-weights)
→ Enter / "see all" → same endpoint with default bound → full grouped view
```

### 8.3 Identifier fast path

```text
GET .../search?q=SHIP-24 → exact non-archived SHIP-24 in-workspace leads issues[0]
→ remaining issue slots + all other groups run normal legs (so "SHIP-24" in a comment still hits)
→ unknown/deleted/archived number ⇒ normal legs only (no match, no error)
```

### 8.4 Short-token path (<2 chars)

```text
GET .../search?q=a → tsquery arm skipped per leg; ILIKE arms carry all five groups
→ still 200 with genuine matches (rule 4); ordering falls back to recency within bound
```

### 8.5 F5 `?q` preservation (non-sequence — the absence of one)

```text
Issues page filters/sort/search/export flows → unchanged F5 contracts, byte-for-byte
→ grouped search shares no code path with list filtering beyond the stemmer config
→ F5 test suite runs unmodified as the proof (api-design §10.1 cross-reference)
```

---

## 9. Module layout

### 9.1 API — `apps/api/src/features/search/`

```text
features/search/
├── routes.ts        # single GET path def → middleware chain → controller; Zod validated at entry
├── schemas.ts       # q/type/limit query coercion (trim, bounds, defaults); shared shapes in packages/shared
├── controller.ts    # HTTP concerns only: parse query, call service, map result/errors
├── service.ts       # fan-out orchestration: per-leg bound resolution (20/50/5), identifier
│                    # short-circuit branch, grouped assembly; NO domain rules of its own
├── repository.ts    # five typed $queryRaw legs (four FTS + one member ILIKE) + exact-identifier lookup
└── errors.ts        # ~empty: WORKSPACE_NOT_FOUND passthrough; leg failures are 500s (no partials)
```

The repository's raw SQL reads other modules' tables — the documented standing exception to the "read via owning service" rule (dashboard §3.2): the indexed columns exist solely for this consumer (every handoff says "F10 implements"), and rank-ordering is inexpressible through the owning list contracts. Returned shapes remain owning card schemas, so the exception is query-plumbing only, never domain logic.

Shared guards reused: `require-session.ts` (F1), `workspace-context.ts` (F2, `rejectArchived: false`). No role guard.

### 9.2 Shared — `packages/shared/src/search/`

`searchTypeSchema`, `searchQuerySchema`, `searchCommentHitSchema`, `searchResultsSchema` (data-model §4). Entity cards re-exported from owning modules.

### 9.3 Web — `apps/web`

| Surface | Route | Reads |
|---|---|---|
| Header search box | App shell (all `/w/*`) | #1 debounced ~250ms (`limit=5` while typing → dropdown; default bound on submit → full view) |
| Grouped results | Overlay → full view | Five groups with counts ("Issues · 4"), empty-group pruning (empty arrays hidden, not errored) |
| "Search within" | Dropdown on the view | `?type=` mapping; bound rises to 50 |
| No-result | View | Designed state with query echo + clear action (spec §4.2) |
| Hit navigation | Per hit | Issue/project/cycle pages, member drawer, comment permalink `#comment-<id>` |

Data access via TanStack Query (`useSearch`, query-keyed on `(q, type)`, stale-while-typing with cancellation of superseded keystrokes). All surfaces ship with loading (skeleton rows), error (retry), empty/no-result, and archived-tolerant states (hits are never archived by construction).

---

## 10. Testing strategy

### 10.1 API integration tests

Supertest + Testcontainers (real GIN behavior — never mocked). Seeded corpus per conventions: issues/projects/cycles/comments/members with overlapping vocabulary ("checkout" in an issue title, a project description, a comment body; "Maya" as member + comment mention).

| Case | Covered by |
|---|---|
| Happy path — grouped `200` with all five arrays bounded | shape + bound assertions |
| Missing `q` key | `400 VALIDATION_ERROR` |
| Blank `q` (`?q=%20`) | `200` all-empty (never error, never dump) |
| `q` > 200 chars | `400` |
| Bad `type`/`limit` | `400` |
| Unauthenticated | `401` |
| Non-member | `404 WORKSPACE_NOT_FOUND` byte-equal (leak test, per leg — assert second workspace's terms never surface) |
| Prefix — "che" hits "checkout" title | prefix assertions per content leg |
| Short token — "a" returns matches via `ILIKE` arms | rule-4 assertions |
| Exact `SHIP-24` leads issues; unknown number ⇒ normal legs, no error | short-circuit assertions |
| Archived issue/project/cycle absent; comment on archived issue absent | exclusion assertions per leg |
| Members — name hit present; email-prefix-only query (`?q=bob@ex`) matches nothing extra | name-only assertions (D5 — supersession proof) |
| Weights — title hit outranks body-only hit for the same term | ordering assertions (D3) |
| Determinism — same query twice ⇒ byte-equal bodies | rule-5 assertions |
| Bounds — 25 seeded issues ⇒ 20 default; `?type=issues` ⇒ 50-cap path; `limit=5` ⇒ 5 | bound assertions |
| `type` filter populates only its group | group-isolation assertions |
| F5 `?q` suite runs unmodified | preservation proof (§8.5) |
| Archived workspace searchable over non-archived rows | tolerance assertions |
| Leg failure (test hook) ⇒ `500`, never partial groups | no-partials assertions |

### 10.2 Component tests — MSW mock of `.../search*`

Box debounces keystrokes (timer mocks assert request coalescing + superseded cancellation); dropdown renders `limit=5` hits; full view groups with counts; `type` dropdown remaps; no-result state with echo + clear; hit click navigates (comment hits include `#comment-<id>`); error retry; archived-tolerant rendering (vacuous by construction — asserted via absence).

### 10.3 End-to-end journey

```text
1. Owner + Maya exist; corpus: issue "Fix checkout redirect", project "Checkout revamp", comment "checkout decided", member Maya
2. Owner types "check" → grouped dropdown → picks issue → detail opens
3. "Search within: Members", "may" → Maya row → member drawer opens
4. "SHIP-2" → exact issue leads; click → detail
5. Archive the issue → same query → issue + its comment gone from results
6. Blank search → no-result state, no error; cross-workspace term → nothing leaks
```

Negatives: missing-`q` direct fetch → 400; non-member slug ⇒ not-found; forced leg failure ⇒ full error view (never partial groups).

---

## 11. Cross-cutting concerns

| Concern | Approach |
|---|---|
| **Keystroke discipline** | Client debounce ~250ms + in-flight cancellation (query-keyed); server rate-guard as backstop (F12). Suggestions and full view share the endpoint — no duplicate infrastructure. |
| **Freshness** | Transactional generation (D1): commit-write ⇒ searchable-read, zero lag, no reindex job. |
| **Performance budget** | Five bounded legs (≤50 each) + GIN matches; member leg over tens of rows; `%..%` containment is the known non-indexed arm — `pg_trgm` iff measured slow (D4). |
| **Pagination** | None — bounds are display caps with drill-down links, not pages (same doctrine as dashboard panels). |
| **Copy/empty states** | `q` echo powers "No results for …" + clear action; empty arrays hidden per group. |
| **Stemming scope** | English-only (arch 12); Meilisearch exit ramp stays documented, evidence-gated (§7 of data-model). |

---

## 12. References

- Shipyard: `features/search/spec.md`, `features/search/data-model.md`, `features/workspace/api-design.md` (context, envelope, read-when-archived), `features/issues/api-design.md` §5.1 (F5 `q` preservation), `features/comments/api-design.md` #2 (permalink convention), `features/members/api-design.md` §11 (directory note), `features/auth/api-design.md` (session), `00-architecture.md` §5–§8 + decision 12, `ADR-001`–`ADR-003`, `Implementation Plan.md` F10
- PostgreSQL full-text search: `https://www.postgresql.org/docs/current/textsearch.html`
- PostgreSQL GIN + `tsvector`: `https://www.postgresql.org/docs/current/textsearch-indexes.html`

---

*Next artifact: implementation (plan §5 Steps 3–7) — Prisma migration scaffold + raw-SQL append (four generated columns + four GIN indexes, automatic backfill) → module code (routes/controller/service/repository with five typed `$queryRaw` legs + shared schemas) → web slice (header box, debounce, grouped view, within-filter, hit navigation) → tests → `pnpm check`. No further design doc needed; data-model + api-design cover F10's technical design.*
