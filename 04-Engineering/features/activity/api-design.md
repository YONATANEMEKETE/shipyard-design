# Activity Log — API Design

**Status:** Draft for review
**Last updated:** 2026-09-04
**Sources:** `features/activity/spec.md` · `features/activity/data-model.md` (locked — `activity_event`, D1–D7) · `features/workspace/api-design.md` (F2 precedent — `:slug` context, read-when-archived, error envelope) · `features/issues/api-design.md` (F5 precedent — cursor pagination, call-site wiring pattern) · `features/dashboard/api-design.md` (F9 consumer — panel migration) · `features/auth/api-design.md` (F1 — Better Auth session) · `00-architecture.md` §5–§8 (cross-module calls via public service APIs) · `ADR-001`–`ADR-003` · `Implementation Plan.md` F9 §7
**Owner:** `apps/api` — hand-written Shipyard code through the canonical pipeline (`route → validation → permission check → controller → service → repository → Prisma`).

---

## 1. Base path & conventions

| Concern | Choice |
|---|---|
| Base path | `/api/v1/workspaces/:slug/activity` — workspace-scoped collection (NOT global like notifications — the log belongs to one workspace's page). Single `GET` list route; emission is internal-only, never HTTP (data-model D2). |
| Next.js proxy | Browser never hits the API directly (ADR-003); `apps/web` forwards `/api/v1/*` → `http://api:4000/api/v1/*`, cookies forwarded. |
| Auth transport | HttpOnly Better Auth session cookie read by `requireSession` (F1). |
| Validation | Zod query schemas from `packages/shared` at the route boundary (`area`, `actorId`, `entityType`, `limit`, `cursor`). No body. |
| Envelope | Success: `{ events: [...], nextCursor }`. Failure: `{ "error": { "code", "message", "details"? } }` via the global error handler. |
| Workspace context | Reuses F2 `resolveWorkspaceContext(:slug)` with `rejectArchived: false` — archived workspaces stay readable. |
| Emission | `activityService.record(event, tx)` — internal writer called by source services in-tx. No `POST` route exists (same rule as notifications D9). |

---

## 2. Endpoint inventory

One endpoint. Emission/retention/dashboard-consumption are **not** endpoints (§5.2). No extras.

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 1 | `GET` | `/api/v1/workspaces/:slug/activity` | `requireSession` → member (any role) | §2/§3.3 Page walk — newest first. Query: `?area=` (one of six, omitted ⇒ all) · `?actorId=` · `?entityType=` · `?limit` (default 25, max 100) · `?cursor`. Allowed while workspace `ARCHIVED`. Deleted-entity rows render frozen (no links, never errors). |

> **Why one route:** the log is an append-only browsed record — no row mutations, no read state, no lifecycle. Everything a reader needs is the filtered walk; everything a writer needs is the internal `record()` call.
>
> **Why no detail route:** rows carry their full frozen summary — there is no "more" to fetch per row. Deep-linking targets the living entity (issue/project/cycle pages), not the event row.

---

## 3. Context resolution

Reuse F2's resolver verbatim: `resolveWorkspaceContext(:slug)` → `{ workspaceId, ... }`, generic `404 WORKSPACE_NOT_FOUND` on miss/non-membership. No role check (any member reads all — spec rule 4). No `:eventId` lookup exists. Filter params (`actorId`, `entityType`) are filters, not scope — unknown values match zero rows (`200` empty, never 404).

---

## 4. Guard chain (canonical, read-only)

```text
requireSession                     ← F1: valid Better Auth session else 401
  │
resolveWorkspaceContext(slug)      ← F2, rejectArchived: FALSE → 404 generic / 200 when archived
  │
controller → service               ← filtered newest-first walk; no role/state preconditions
```

No `requireWorkspaceRole`, no authorship, no confirmations — the only write path is internal emission inside source transactions.

---

## 5. Request/response contracts

### 5.1 Page walk

| Endpoint | Query | Success response |
|---|---|---|
| #1 list | `?area=workspace\|members\|projects\|issues\|comments\|cycles` (maps to kind sets server-side) · `?actorId=<cuid>` · `?entityType=...` · `?limit=1..100` (default `25`) · `?cursor=<opaque>` | `200` + `{ events: activityEventCardSchema[], nextCursor: string \| null }` (`createdAt DESC, id DESC`); `nextCursor null` ⇒ end |

- Bad `limit`/`cursor`/unknown `area`/`entityType`/non-cuid `actorId` ⇒ `400 VALIDATION_ERROR`. Unknown `actorId` (valid cuid, no rows) ⇒ `200` empty.
- Frozen rows (deleted entities, nulled actors) return verbatim — `entityTitle`/`actorName` render without links. The page never 404s for row state.

### 5.2 Non-endpoints — emission call sites (in-tx service calls)

| Caller tx | Call | Row |
|---|---|---|
| Workspace rename/icon/archive/restore | `record({kind: WORKSPACE_* , entity: WORKSPACE/slug, ...})` | summary from frozen names |
| Invite / accept / decline / revoke / remove / leave / role change / ownership transfer | `record({kind: MEMBER_* / OWNERSHIP_TRANSFERRED, ...})` | invitee email / member name frozen |
| Project CRUD/lifecycle/transfer | `record({kind: PROJECT_*, entity: PROJECT/id, ...})` | project name frozen |
| Issue create/status/assign/block/archive/restore/delete | `record({kind: ISSUE_*, entity: ISSUE/id, ...})` | `"SHIP-24 · title"` frozen; dual-written with `issue_history` (D7) |
| Comment create/delete | `record({kind: COMMENT_*, entity: COMMENT/id, ...})` | issue ref frozen; edits emit nothing |
| Cycle lifecycle | `record({kind: CYCLE_*, entity: CYCLE/id, ...})` | cycle name + dates frozen |

All execute inside the caller's `$transaction` (strict — failure rolls back the source, D2). Full per-route wiring table lands at implementation; the kind enum (§2.1 of data-model) is the checklist.

---

## 6. Archived / state matrices

| Situation | #1 list |
|---|---|
| Workspace `ARCHIVED` | ✅ allowed (log frozen by upstream silence, not by guard) |
| Event references deleted entity | ✅ frozen text, no link |
| Event actor deleted | ✅ frozen `actorName`, no member link |
| Empty log (fresh workspace) | ✅ `200` `{ events: [], nextCursor: null }` + designed empty state |

---

## 7. Error codes (Activity module)

| Code | HTTP | When | Notes |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | Bad `limit`/`cursor`/`area`/`entityType`/`actorId` | `details` lists field paths |
| `WORKSPACE_NOT_FOUND` | 404 | Unknown `:slug` or caller not a member — identical | No existence leak |
| `UNAUTHENTICATED` | 401 | Missing/expired session cookie | F1 |
| `RATE_LIMITED` | 429 | Global limiter (F12) | `Retry-After` |

No `FORBIDDEN_ROLE`, no `*_ARCHIVED` 409s, no `NOT_FOUND` for filter misses or frozen rows. No `POST` route exists to confirm — route-table tests assert it 404s (same discipline as notifications §10.1).

---

## 8. Sequences

### 8.1 Page walk

```text
Member → GET .../activity?limit=25 → { events: [...25 newest], nextCursor }
→ filter chips: GET .../activity?area=members → member-kinds subset walk (own cursor chain)
→ actor filter: GET .../activity?actorId=<cuid> → that human's actions
→ click living entity → its page; click deleted entity → frozen text, no navigation
```

### 8.2 Strict emission (representative — decline flow from the user's example)

```text
Bob → POST .../invitations/:token/decline → members service tx {
  UPDATE invitation status=DECLINED
  record({ kind: MEMBER_DECLINED, entity: INVITATION/token-id,
           actor: invitee userId + email (D4 — invitee is not a member),
           summary: "bob@example.com declined the invite to Harbor" }, tx)
} → 200 → activity page shows invite + decline as consecutive rows
→ forced record() failure in tests ⇒ decline also rolls back (strict, D2)
```

### 8.3 Delete-event survival (the D3 proof)

```text
Owner → DELETE .../projects/:id {confirmName} → projects service tx {
  ... unassign issues, DELETE project ...
  record({ kind: PROJECT_DELETED, entity: PROJECT/id,
           entityTitle: "Ship Payroll", summary: "Owner deleted project Ship Payroll" }, tx)
} → project gone; its log rows (created/renamed/deleted) all remain, rendered frozen
```

### 8.4 Dashboard migration (F9 §7 consumer)

```text
Dashboard service (post-landing): activityFeed(ws) →
  activityService.listRecent(ws, kinds=[issue/comment kinds], limit=20)
  // replaces the issue_history+comments union; same bound, same card mapping
→ pre-log events absent (accepted gap); entity timelines cover the past
```

---

## 9. Module layout

### 9.1 API — `apps/api/src/features/activity/`

```text
features/activity/
├── routes.ts        # single GET path def → middleware chain → controller
├── schemas.ts       # area/actorId/entityType/limit/cursor query coercion; shared shapes in packages/shared
├── controller.ts    # HTTP concerns only
├── service.ts       # page-walk query + internal record()/listRecent() writers/readers
├── repository.ts    # Prisma access only (activity_event reads/inserts)
└── errors.ts        # ~empty: WORKSPACE_NOT_FOUND passthrough
```

Cross-module contracts (this module owns the receiving side):

```text
activityService.record(event, tx)            → called by workspace/members/projects/issues/comments/cycles in-tx
activityService.listRecent(workspaceId, kinds, limit) → called by dashboard service (F9 migration)
```

### 9.2 Shared — `packages/shared/src/activity/`

`activityKindSchema`, `activityEntityTypeSchema`, `activityAreaSchema`, `recordActivityEventSchema` (internal), `activityEventCardSchema`, `activityListQuerySchema`, `activityListPageSchema`.

### 9.3 Web — `apps/web`

| Surface | Route | Reads |
|---|---|---|
| Activity page | `/w/:slug/activity` (+ `?area=&actorId=`) | #1 walk; area chips, actor filter, frozen-row rendering (title without link), empty state |
| Dashboard feed | hub (post-migration) | Unchanged panel contract, new source (§8.4) |

TanStack Query standard query + `fetchMore` on `nextCursor`; load on navigation only (no polling). Loading skeletons, error retry, empty (quiet workspace), archived-tolerant states.

---

## 10. Testing strategy

### 10.1 API integration tests

Supertest + Testcontainers. Seeded helpers per existing conventions.

| Case | Covered by |
|---|---|
| Happy path — walk with filters (area/actor/entityType), cursor to `nextCursor: null` | ordering + filter + cursor assertions |
| Invalid query (bad area/limit/cursor/actorId) | `400 VALIDATION_ERROR` |
| Unauthenticated | `401` |
| Non-member | `404 WORKSPACE_NOT_FOUND` byte-equal (leak test) |
| No `POST` route — any body | `404` route-table assertion |
| Emission per area (representative + full kind checklist over time) | cross-module tests asserting rows + frozen summaries |
| Strictness — forced `record()` failure rolls back source | rollback assertions (invite/issue/cycle reps) |
| Delete survival — project/issue/comment delete keeps rows, frozen titles render | survival + verbatim assertions |
| Actor delete — rows keep frozen name, `actorId` nulled | fallback assertions |
| Exclusions — label ops, comment edits, resends emit nothing | row-count 0 assertions |
| Archived workspace readable; empty log `200` empty | no-freeze + empty assertions |
| Unknown filter values ⇒ `200` empty (never 404) | filter-not-scope assertions |
| Dashboard `listRecent` returns issue/comment kinds ≤20 newest-first | consumer-contract test |

### 10.2 Component tests — MSW mock of `.../activity*`

Page renders frozen/event cards newest-first; area chips refetch; actor filter narrows; deleted-entity rows show title without links; empty state; error retry; archived-tolerant rendering.

### 10.3 End-to-end journey

```text
1. Owner invites Bob → log: "invited bob@…"; Bob declines → log: "declined" (the user's example, end to end)
2. Owner creates project + issue, moves issue, comments, starts cycle → each narrated in order
3. Member filters to members-area → invite/decline rows only
4. Owner deletes the issue → its rows remain frozen; clicking shows text, no link
5. Archive workspace → page still readable
```

Negatives: non-member slug ⇒ not-found; `POST` attempt ⇒ 404; forced emission failure ⇒ source rolled back (integration-level).

---

## 11. Cross-cutting concerns

| Concern | Approach |
|---|---|
| **Rate limiting** | Global limiter only — single idempotent `GET`; emission bounded by source-route caps. |
| **Freshness** | Load-on-navigation + focus refetch; no polling, no realtime (arch §11). |
| **Pagination** | Cursor only (`(createdAt, id)` DESC). Default 25, max 100. |
| **Copy ownership** | Server-rendered frozen `summary` (D5) — page renders verbatim; no client presenter. |
| **Search** | No `q` — kind/actor/entity filters only (§7 of data-model). |
| **Retention** | Uncapped (D6); revisit with evidence. |

---

## 12. References

- Shipyard: `features/activity/spec.md`, `features/activity/data-model.md`, `features/workspace/api-design.md` (context, envelope, read-when-archived), `features/issues/api-design.md` (cursor pagination, call-site pattern), `features/dashboard/api-design.md` (consumer panel), `features/notifications/api-design.md` (internal-emission precedent, no-create-route discipline), `features/auth/api-design.md` (session), `00-architecture.md` §5–§8, `ADR-001`–`ADR-003`
- Prisma: `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- Cursor pagination: `https://www.prisma.io/docs/orm/prisma-client/queries/pagination#cursor-based-pagination`

---

*Next artifact: implementation (plan §5 Steps 3–7) — Prisma migration (`activity_event` + enums + back-relations, no raw SQL, no backfill) → module code (routes/controller/service/repository + `record()`/`listRecent()` internals) → per-area call-site wiring (workspace/members/projects/issues/comments/cycles) → dashboard feed migration → web slice (activity page, filters, frozen rendering) → tests → `pnpm check`.*
