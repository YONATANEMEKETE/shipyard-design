# Notifications — API Design

**Status:** Draft for review
**Last updated:** 2026-09-04
**Sources:** `features/notifications/spec.md` · `features/notifications/data-model.md` (locked — `notification` + `NotificationType`, D1–D9) · `features/workspace/api-design.md` (F2 precedent — error envelope, leak-free 404s) · `features/members/api-design.md` (F3 precedent — `confirm: true` convention) · `features/issues/api-design.md` (F5 precedent — cursor pagination, assignment call-site) · `features/comments/api-design.md` (F8 precedent — mention call-site, retraction contract) · `features/auth/api-design.md` (F1 — Better Auth session) · `00-architecture.md` §5–§8, §11 (polling, no WebSockets) · `ADR-001`–`ADR-003` · `Implementation Plan.md` F6

> **Principle:** identical pipeline, inverted scope — every route is hand-written Shipyard code through the canonical chain:
>
> ```text
> route → validation → recipient check → controller → service → repository → Prisma
> ```
>
> Better Auth handles identity; this module owns **recipient isolation** — a caller touches only rows where `recipientId = session.userId`. There is no workspace context, no role check, and no create route: rows are born only inside source transactions (data-model D9) and die by recipient action or source cascade.
>
> **The D2 divergence, stated once:** these are the only global (non-`:slug`) routes in the MVP outside auth/invitations. The bell follows the human across workspaces, so `resolveWorkspaceContext` never runs here — recipient ownership replaces it.

---

## 1. Base path & conventions

| Concern | Choice |
|---|---|
| Base path | `/api/v1/notifications` and `/api/v1/notifications/:notificationId` — global, no `:slug` (D2). Collection reads, per-row reads/mutations, and bulk actions all live here. |
| Next.js proxy | Browser never hits the API directly (ADR-003); `apps/web` forwards `/api/v1/*` → `http://api:4000/api/v1/*`, cookies forwarded. |
| Auth transport | HttpOnly Better Auth session cookie read by `requireSession` (F1) — `req.session.userId` is the only identity input and the only scope key. |
| Validation | Zod schemas from `packages/shared` (`data-model.md` §4) at the route boundary. Cursor is opaque base64url of `(createdAt, id)`. |
| Envelope | Success: resource JSON directly (or `{ notifications: [...], nextCursor }` / `{ unreadCount }` / `{ markedCount }` / `{ deletedCount }`). Failure: `{ "error": { "code", "message", "details"? } }` via the global error handler. |
| Scope | Recipient scope — every query carries `WHERE recipientId = session.userId`. No workspace lookup, no membership check, no role check. A foreign `notificationId` is indistinguishable from a random cuid (`404 NOTIFICATION_NOT_FOUND`, §3). |
| Workspace archiving | Irrelevant at the guard layer (no workspace context to freeze). Reads always allowed; recipient mutations always allowed; new rows cannot arise in archived workspaces because source writes are frozen upstream (data-model §6.7). |
| Polling | `GET unread-count` is the ~60s poll target (partial-index cheap); the panel refetches on open/focus, not on a timer (arch §11). |

---

## 2. Endpoint inventory

Seven endpoints cover every behavior in `spec.md` §2–§5. Emission and retraction are **not** endpoints (in-tx service calls, §5.2). No extras.

### 2.1 Global — panel, badge, row actions, bulk actions

All under `/api/v1/notifications...`, all through the §4 recipient chain.

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 1 | `GET` | `/api/v1/notifications` | `requireSession` → recipient scope | §2/§3.2 Panel — newest first. Query: `?unreadOnly=true` (default false), `?workspaceId=<cuid>` (optional per-workspace filter), `?limit` (default 25, max 100), `?cursor`. Cards carry live issue + actor + `commentId` (deep-link context). |
| 2 | `GET` | `/api/v1/notifications/unread-count` | `requireSession` → recipient scope | §2 Badge — `{ unreadCount }` over `readAt IS NULL` via the partial index. The poll loop hits only this. |
| 3 | `GET` | `/api/v1/notifications/:notificationId` | `requireSession` → recipient scope | §2 Detail — one card (permalink validation before navigation). Foreign/unknown id ⇒ `404`. |
| 4 | `POST` | `/api/v1/notifications/:notificationId/read` | `requireSession` → recipient scope | §2 Mark read — idempotent (`readAt` keeps its first timestamp). Empty body. Already-read ⇒ `200` unchanged. |
| 5 | `POST` | `/api/v1/notifications/read-all` | `requireSession` → recipient scope | §2 Mark all read — one statement over `readAt IS NULL`, optional `?workspaceId=` scope. Body `{}`. Returns `{ markedCount }` (0 is a valid success). |
| 6 | `DELETE` | `/api/v1/notifications/:notificationId` | `requireSession` → recipient scope | §2 Delete one — permanent. Body `{ confirm: true }` (same convention as labels/cycles/comments deletes). Returns `{ deletedNotificationId }`. |
| 7 | `DELETE` | `/api/v1/notifications` | `requireSession` → recipient scope | §2 Clear all — permanent. Body `{ confirm: true }`, optional `?workspaceId=` / `?readOnly=` (`readOnly=true` clears read rows only). Returns `{ deletedCount }`. |

> **Why `POST …/read` instead of `PATCH {read:true}`:** read-state is a one-way idempotent action, not a resource edit — there is no `read:false` transition in MVP (data-model D6), so a PATCH boolean would advertise a transition that 409s. The `POST` verb makes the one-waydoorn explicit, same reasoning as lifecycle actions in cycles (`POST …/archive` over `PATCH {archived:true}`).
>
> **Why no `POST /notifications`:** rule 7 forbids sourceless events (data-model D9). The only writers are `createAssignment`/`createMention` inside source txs (§5.2) — a public create would mint events no issue/comment owns. Route-table tests assert the path 404s (§10.1).
>
> **Why count is separate from the panel:** the badge polls every ~60s all app-open; coupling it to the 25-row panel query would fetch cards at polling frequency for a number. `unread-count` is a covered-index `COUNT(*)` (§8.1); the panel loads on open/focus only.

---

## 3. Context resolution

### 3.1 Recipient scope — the only lookup (all routes)

```text
req.session.userId ──recipient scope──▶ WHERE recipientId = session.userId (+ id for #3/#4/#6)
```

- No workspace resolution, no membership check, no role check. The session *is* the scope.
- Unknown `:notificationId` **or** another recipient's row ⇒ identical `404 NOTIFICATION_NOT_FOUND` — the inverted leak test: assert a foreign id and a random cuid return byte-equal bodies (mirrors F2's slug leak test, applied to recipient isolation).
- `?workspaceId=` on #1/#5/#7 is a **filter**, not a scope lookup: unknown or foreign workspace ids simply match zero rows (`200` with empty set / zero counts — never 404). No membership validation runs on it, so no existence signal leaks through the filter.
- Detail (#3) re-resolves the live issue join for navigation (`issueId → issue` + `workspace.slug`); a notification whose issue vanished cannot be observed — cascade deletes it first (rule 5), so the join never dangles.

### 3.2 Source-side resolution (emission — not routes, documented for the call sites)

Owned by the source services inside their transactions: issues resolves `(workspaceId, issueId, newAssigneeId, actorId)` with actual-change + non-self checks; comments resolves the deduped recipient list from `comment_mention` rows minus the author. F6 validates recipient-liveness (user row still exists) defensively in-tx and skips dead recipients without failing the source write.

---

## 4. Guard chain (canonical — recipient variant)

### 4.1 All notification routes (#1–#7)

```text
requireSession                     ← F1: valid Better Auth session else 401
  │
recipientScope                     ← WHERE recipientId = session.userId on every query
  │                                  (collection filters, per-row lookups, bulk statements)
  ├─ notificationById :notificationId ← (#3/#4/#6) scoped lookup → 404 NOTIFICATION_NOT_FOUND
  │                                            on miss OR foreign row (no leak)
  │
controller → service               ← read/delete preconditions (already-read idempotency,
                                      confirm:true) — no role, no workspace state, ever
```

What this chain deliberately omits (vs F2–F8): no `resolveWorkspaceContext`, no `requireWorkspaceRole`, no `rejectArchived`, no authorship check (recipient *is* the authorizer). Service reasserts `recipientId` on writes even though scoping already ran — defense in depth against a future route forgetting the filter.

---

## 5. Request/response contracts

Schemas from `packages/shared` (`data-model.md` §4). Route handlers validate bodies, params, and query before anything else.

### 5.1 Reads & mutations

| Endpoint | Body / Query | Success response |
|---|---|---|
| #1 panel | query `?unreadOnly=true\|false` (default `false`) · `?workspaceId=<cuid>` (optional filter — unknown matches zero rows, §3.1) · `?limit=1..100` (default `25`) · `?cursor=<opaque>` | `200` + `{ notifications: notificationCardSchema[], nextCursor: string \| null }` newest-first (`createdAt DESC, id DESC`); `nextCursor null` ⇒ end |
| #2 unread-count | — (no query) | `200` + `{ unreadCount: number }` |
| #3 detail | — | `200` + `notificationCardSchema` |
| #4 mark read | `{}` (empty body suffices; id is in the path) | `200` + `notificationCardSchema` (`readAt` set; already-read returns the unchanged card) |
| #5 mark all read | `{}` · query `?workspaceId=<cuid>` (optional scope) | `200` + `{ markedCount: number }` |
| #6 delete one | `{ confirm: true }` | `200` + `{ deletedNotificationId }` |
| #7 clear all | `{ confirm: true }` · query `?workspaceId=<cuid>` (optional scope) · `?readOnly=true` (optional — clears read rows only; default false clears everything) | `200` + `{ deletedCount: number }` |

Validation & precondition details:

- `#1`: bad `limit`/`cursor`/non-cuid `workspaceId` ⇒ `400 VALIDATION_ERROR`. Cursor is bound to the `(createdAt, id)` DESC order — the only order this endpoint serves. `unreadOnly=true` composes with cursor (walk the unread subset to its own `nextCursor: null`).
- `#2`: never 404s — zero is `{ unreadCount: 0 }`. No query params (unknown params ⇒ `400 VALIDATION_ERROR`, same strictness as issue-list unknown pagination params).
- `#4`: empty body required-but-ignored beyond shape; foreign/already-deleted id ⇒ `404 NOTIFICATION_NOT_FOUND` (indistinguishable). Read is one-way: no unread transition exists — a body attempting one ⇒ `400 VALIDATION_ERROR`.
- `#5`: `markedCount: 0` (nothing unread) is `200`, not an error. `workspaceId` scoping is a filter on the `UPDATE` — same zero-match semantics as #1.
- `#6`/`#7` require literal `confirm: true` (missing ⇒ `400 CONFIRMATION_REQUIRED`, same precedent as labels/cycles/comments). `#7` without filters clears the whole inbox — the client dialog states the blast radius; the API enforces its own minimum via the literal.
- `#6` on an already-deleted id ⇒ `404 NOTIFICATION_NOT_FOUND` (gone reads as never-there, matching comment-delete tombstone-less semantics).

### 5.2 Non-endpoints — emission & retraction (in-tx service calls)

| Call | Caller tx | Effect |
|---|---|---|
| `createAssignment({ workspaceId, issueId, newAssigneeId, actorId }, tx)` | Issues create-with-assignee / actual-change reassign | `INSERT` one `ASSIGNMENT` row (`commentId: NULL`). Caller pre-filters same-person/unassign/self (D8). |
| `createMention({ workspaceId, issueId, commentId, recipientId, actorId }, tx)` | Comment create, per distinct recipient minus author | `INSERT` one `MENTION` row per call. Caller dedupes via the `comment_mention` PK. |
| `deleteForComment(commentId, tx)` | Comment delete / issue delete | `DELETE WHERE commentId = ?` (retraction, D4). FK Cascade covers it where modeled; the call is the intent-readable path. |

None are reachable over HTTP — exposed only on the notifications service for cross-module import (same pattern as F3→F4 `transferOwnedProjects`).

---

## 6. Read-only / state matrices

### 6.1 Workspace & issue state — no axis here

Unlike every prior module, notifications has **no archived matrix**: there is no workspace context to freeze and no notification-level archive dimension. Concretely:

| Situation | Panel/badge/detail | Mark read/all | Delete/clear |
|---|---|---|---|
| Owning workspace `ARCHIVED` | ✅ allowed | ✅ allowed | ✅ allowed |
| Related issue archived | ✅ allowed (read-only landing, spec §3.2) | ✅ allowed | ✅ allowed |
| Related issue deleted | n/a — rows already cascaded (rule 5) | n/a | n/a |

New rows cannot reference frozen containers because source writes are frozen upstream (data-model §6.7) — F6 asserts defensively and skips emission rather than erroring.

### 6.2 Row-state matrix (recipient scope, active session)

| Endpoint | Unread row | Read row | Foreign/missing row |
|---|---|---|---|
| #1/#3 read | ✅ | ✅ | ❌ `404 NOTIFICATION_NOT_FOUND` |
| #2 count | counted | ignored | n/a |
| #4 mark read | ✅ sets `readAt` | ✅ no-op `200` (first `readAt` kept) | ❌ `404` |
| #5 mark all | ✅ counted in `markedCount` | ignored | n/a (zero-match ⇒ `markedCount: 0`) |
| #6/#7 delete | ✅ removed | ✅ removed (read state never blocks deletion) | ❌ `404` (#6) / ignored in count (#7) |

---

## 7. Error codes (Notifications module)

Global error handler converts typed domain errors; controllers never build envelopes by hand.

| Code | HTTP | When | Notes |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | Zod body/param/query failure (bad `limit`/`cursor`/`workspaceId`, non-empty #4/#5 bodies, bad `readOnly` flag) | `details` lists field paths |
| `CONFIRMATION_REQUIRED` | 400 | #6/#7 without literal `confirm: true` | Same precedent as labels/cycles/comments |
| `NOTIFICATION_NOT_FOUND` | 404 | `:notificationId` unknown **or** owned by another recipient — deliberately identical | Inverted leak test (§3.1): assert byte-equal to random-cuid |
| `UNAUTHENTICATED` | 401 | Missing/expired session cookie | F1 `requireSession` |
| `RATE_LIMITED` | 429 | Poll-guard on #2 (wiring finalized at F12; global limiter exists) | `Retry-After` header |

No `WORKSPACE_NOT_FOUND` (no workspace scope), no `FORBIDDEN_ROLE` (no roles — recipient isolation replaces them), no `ALREADY_*`/`NOT_ARCHIVED` (no lifecycle dimension), no `*_ARCHIVED` 409s (no freeze axis, §6.1). Unknown `?workspaceId=` filter values are **not** errors — they match zero rows (§3.1).

---

## 8. Sequences

### 8.1 Badge poll loop (the ~60s heartbeat, spec §4.4)

```text
App open → every ~60s (TanStack Query, decorator pauses when tab hidden):
  GET /api/v1/notifications/unread-count → 200 { unreadCount: 3 }
→ header bell renders badge (0 ⇒ dot hidden, no panel fetch)
→ failure → keep last count + silent retry (never an error banner for the poll)
→ panel open / window focus → GET #1 (fresh walk) instead of relying on the count
```

### 8.2 Panel walk (newest-first cursor)

```text
GET /api/v1/notifications?limit=25 → { notifications: [...25 newest], nextCursor: "ey..." }
GET ...?cursor=ey... → next 25 → ... → { ..., nextCursor: null } ⇒ end
GET ...?unreadOnly=true → unread subset walk (own cursor chain to null)
→ click row → GET #3 (validate) → navigate per §8.5 → POST #4 (mark read) → badge decrements on next poll
→ opening marks read; it never deletes (spec §3.2 — read rows persist until dismissed)
```

### 8.3 Assignment emission (F5 ↔ F6 — the call sites this design serves)

```text
Maya → POST/PATCH issue (assignee: Bob) → issues guard (any member) → issues service tx {
  assert actual change (old !== new, new != null) else no-op (no write/history/event)
  assert new !== actor else skip emission (D8 self-suppress — row writes normally)
  ... issue write + ASSIGNED history ...
  createAssignment({ workspaceId, issueId, newAssigneeId: Bob, actorId: Maya }, tx)
    → INSERT notification { recipient: Bob, actor: Maya, issue, type: ASSIGNMENT }
} → Bob's next poll: unreadCount +1 → panel shows "Maya assigned you to SHIP-024"
→ unassignment (new = null) → no call (spec §3.1 — no unassignment rows, ever)
```

### 8.4 Mention emission + retraction (F8 ↔ F6)

```text
Author → POST comment "@maya look" → comments guard → comments service tx {
  INSERT comment + comment_mention rows ([maya], self excluded from fan-out)
  createMention({ workspaceId, issueId, commentId, recipientId: Maya, actorId: Author }, tx)
} → Maya's badge +1 → click → issue detail scrolled to #comment-<id> (comment permalink, F8 #2)

Author → DELETE comment {confirm:true} → comments service tx {
  DELETE comment → cascades joins
  deleteForComment(commentId, tx) → Maya's mention row for this comment gone (D4, no dead link)
} → Maya's badge decrements on next poll; sibling notifications untouched

Author → PATCH comment (adds @carol) → joins recomputed, zero F6 calls → Carol gets nothing (rule 4)
```

### 8.5 Navigation targets (spec §3.2)

```text
ASSIGNMENT row → /w/:workspaceSlug/issues/:issueId            (issue detail)
MENTION row    → /w/:workspaceSlug/issues/:issueId#comment-<commentId>  (detail + scroll)
  → issue archived ⇒ same URLs land read-only (banner, no composer)
  → issue deleted ⇒ row already cascaded; a stale client cache hitting #3 gets 404 → panel refetches
  → comment deleted ⇒ row already retracted; same 404 → refetch path
```

### 8.6 Issue delete cascade (F5 #7 → F6, closes the F5 leg)

```text
Owner/Admin → DELETE .../issues/:issueId → issues service tx {
  DELETE comment WHERE issueId (F8 leg → cascades joins; deleteForComment per commentId)
  DELETE notification WHERE issueId (direct leg — assignment rows + any stragglers)
  DELETE issue (cascades labels/history per F5 §6.5)
} → recipients' next poll: counts drop; panel rows for the issue gone with it (rule 5)
```

### 8.7 Mark-all-read + clear-all (bulk, recipient-only)

```text
User → POST .../read-all {} → UPDATE ... SET readAt=now() WHERE recipient AND readAt IS NULL
  → 200 { markedCount: 14 } → badge 0 on next poll; rows persist for history
User → DELETE .../ {confirm:true} → DELETE WHERE recipient → 200 { deletedCount: 41 }
  → badge 0 AND panel empty; irreversible (rule 3)
User → DELETE .../?readOnly=true {confirm:true} → clears read rows only; unread badges untouched
```

---

## 9. Module layout

### 9.1 API — `apps/api/src/features/notifications/`

```text
features/notifications/
├── routes.ts        # router: global path defs → recipient-scope → controller; Zod validated at entry
│                    # (#1–#7; NO create route — §5.2 calls live on the service only)
├── schemas.ts       # route-local param/query coercion (notificationId param, unreadOnly/workspaceId/
│                    # limit/cursor query, confirm:true bodies); shared shapes in packages/shared
├── controller.ts    # HTTP concerns only: parse req/query, call service, map result/errors
├── service.ts       # recipient-scoped reads/mutations + internal writers (createAssignment,
│                    # createMention, deleteForComment) called by issues/comments in-tx; transactions
├── repository.ts    # Prisma access only (recipient-scoped panel/count/row/bulk queries, inserts)
└── errors.ts        # typed domain errors → global handler maps to §7
```

Shared guards reused (partially — the documented divergence):

```text
common/guards/
├── require-session.ts           # (F1) — the only guard on every route here
└── workspace-context.ts         # NOT USED in this module (D2) — listed to make the absence reviewable
```

Cross-module contracts (this module owns the receiving side):

```text
notificationsService.createAssignment(event, tx)  → called by issuesService on actual-change assign
notificationsService.createMention(event, tx)     → called by commentsService on comment create
notificationsService.deleteForComment(commentId, tx) → called by commentsService on comment delete
                                                     (+ covered by FK Cascade; call is intent-readable)
```

### 9.2 Shared — `packages/shared/src/notifications/`

Re-exports from `data-model.md` §4 — the canonical place:

- Enum: `notificationTypeSchema`
- Internal (never HTTP): `assignmentEventSchema`, `mentionEventSchema`
- Response: `notificationActorCardSchema`, `notificationIssueCardSchema`, `notificationCardSchema`, `notificationListQuerySchema`, `notificationListPageSchema`, `unreadCountSchema`, bulk-result schemas (`{ markedCount }`, `{ deletedCount }`, `{ deletedNotificationId }`)

Copy is client-side: a `notificationCopy(type, actor, issue)` presenter lives in `apps/web`, not in shared — the API ships facts, never sentences (spec Q1).

### 9.3 Web — `apps/web`

| Surface | Route | Reads/Writes |
|---|---|---|
| Header bell + badge | App shell (all `/w/*` + global) | #2 poll (~60s, pause when hidden); badge from `unreadCount` only |
| Notification panel | Global overlay (any route) | #1 walk (newest-first + `unreadOnly` toggle + optional workspace filter), #4 on open-click, #5 mark-all, #6 per-row dismiss, #7 clear-all |
| Deep-link landing | `/w/:slug/issues/:issueId` (`#comment-<id>` for mentions) | #3 validation before scroll; archived banner when `issue.archivedAt` set; stale-row 404 → panel refetch |
| Empty states | Panel | No notifications; no unread (all caught up); filter-match-empty |

Data access via TanStack Query: `useUnreadCount` (polling query, 60s, hidden-pause) + `useNotifications` (panel query, fetch-on-open/focus, cursor `fetchMore`). Mutations pessimistic for read/delete/bulk (authoritative — badge reconciles on next poll, with immediate local decrement for responsiveness rolled back on failure).

All surfaces ship with loading, error, empty (no notifications / all read), and archived-tolerant states (rows for archived issues render with a read-only hint, never hidden).

---

## 10. Testing strategy

Three layers (mirrors issues/comments §10). Tooling provisioned by F1/F2; no new deps.

### 10.1 API integration tests

Supertest against `createApp()`, real Postgres via Testcontainers + migrations (incl. `notification_unread_idx`). Seeded helpers: `createVerifiedUser`, `createWorkspaceAs(owner)`, `addMember(workspace, user, role)`, `createIssue(workspace, overrides)`, `createComment(issue, author, overrides)`.

| Case | Covered by |
|---|---|
| Happy paths ×7 endpoints | Supertest suite per group (panel, badge, row, bulk) |
| No create route — `POST /notifications` (any body) | `404` route-table assertion (rule 7 — sourceless events unmintable) |
| Invalid input (bad `limit`/`cursor`/`workspaceId`/`readOnly`, non-empty #4/#5 bodies, bad cuid id) | `400 VALIDATION_ERROR` |
| Missing `confirm: true` (#6/#7) | `400 CONFIRMATION_REQUIRED` |
| Unauthenticated ×7 | `401 UNAUTHENTICATED` |
| Foreign-row access (Bob's id under Alice's session; random cuid) | `404 NOTIFICATION_NOT_FOUND` — assert byte-equal (inverted leak test, §3.1) |
| Owner cannot read/mark/delete another's rows (role is irrelevant here) | `404` on every per-row route (recipient isolation, rule 6) |
| Assignment — create-with-assignee emits one `ASSIGNMENT` | cross-module test (calls issues route, asserts row) |
| Assignment — actual-change reassign emits; same-person emits nothing; unassign emits nothing | cross-module tests (row-count assertions) |
| Assignment — self-assign emits nothing (D8) | row-count 0, issue row normal |
| Mention — distinct recipients each get one; duplicates collapse; unknown/self emit nothing | cross-module tests via comments routes (join + row counts) |
| Edit comment emits nothing | row-count unchanged after `PATCH` |
| Comment delete retracts its mention rows only (siblings + assignments survive) | row-count assertions |
| Issue delete removes its assignment + mention rows | cross-module test (calls issues `DELETE`, asserts rows gone) |
| Workspace delete removes its rows | cascade assertion (all recipients) |
| Recipient user cleanup removes inbox; actor cleanup nulls `actorId` (row survives, fallback renders) | FK-behavior assertions |
| Mark read — sets `readAt`, re-mark no-op keeps first timestamp | `200` + DB assertions |
| Mark all — `markedCount` exact, already-read untouched, `0` ⇒ `200` | DB + count assertions |
| Mark all + clear-all with `?workspaceId=` scope only that workspace's rows | scoped-count assertions |
| Clear-all `?readOnly=true` keeps unread (badge unchanged), clears read | count + badge assertions |
| Delete one — row gone; second delete ⇒ `404` | `200` then `404` |
| Archived workspace rows — panel/badge/read/delete all `200` | no-freeze assertions (§6.1) |
| Archived-issue rows — readable, navigable (`archivedAt` on the card), actionable | card-shape + `200` assertions |
| `?workspaceId=` unknown/foreign ⇒ `200` empty (never 404) | filter-not-scope assertions |
| Panel — newest-first order, cursor walk to `nextCursor: null`, invalid-cursor 400 | ordering + cursor assertions |
| `unreadOnly=true` walk covers exactly the unread set | set-equality assertions |
| Badge — equals unread count after every mutation sequence above | after-each assertions (badge is derived, never stored) |
| Emission atomicity — force notification insert failure in test hook | source row also absent (full rollback, rule 7) |

### 10.2 Component tests (web) — MSW mocks of `/api/v1/notifications*`

| Surface | Cases |
|---|---|
| Bell + badge | Renders `unreadCount`; 0 hides the dot; poll interval wired (~60s, hidden-pause asserted via timer mocks); poll failure keeps last count silently |
| Panel | Renders `notificationCardSchema` newest-first; empty states (none / all-read / filter-empty); `unreadOnly` toggle refetches; workspace filter narrows; "load more" walks `nextCursor` |
| Row copy | `notificationCopy()` renders assignment vs mention sentences from `type+actor+issue` (never server sentences); actor-null renders "Former member"; archived-issue rows show read-only hint |
| Open-click | Sends `POST #4`; navigates to issue (mention: `#comment-<id>` scroll); already-read click still navigates; `404` (retracted/deleted source) refetches panel + shows "no longer available" |
| Mark-all / dismiss / clear | Send `#5`/`#6`/`#7` with `confirm:true` (clear); missing-confirm blocked client-side; `markedCount`/`deletedCount` reflected immediately, rolled back on failure |
| Deep-link landing | Validates via `#3`; stale id shows removed-state, never a crash |
| Error envelope rendering | Every surface renders MSW-served `{error:{code,message}}` as friendly states, never raw dumps |

Rules: components never re-implement rules (e.g., "unread = `readAt == null`" is API-shaped; web just renders). Tests assert wire behavior + rendered state.

### 10.3 End-to-end journey — golden path

Playwright against the composed stack (web + api + Postgres, reset between runs). Poll interval shortened via test env for the badge assertions.

**Journey — attention lifecycle (core)**

```text
1. Owner + Maya (Member) exist in a workspace with an issue (F3/F5)
2. Owner assigns the issue to Maya → Maya's badge goes 0 → 1 on next poll
3. Maya opens the panel → sees "Owner assigned you to SHIP-1" → clicks → lands on the issue
4. Opening marked it read → badge back to 0; row persists (read, not gone)
5. Maya comments "@owner look" → Owner's badge → 1 with a mention row
6. Owner clicks the mention → lands scrolled to the comment; marks all read → badge 0
7. Maya deletes her comment (confirm) → Owner's mention row retracts (panel refetch: gone)
8. Owner reassigns the issue to themselves → no new row for Owner (self-suppress)
9. Owner deletes the issue → all its rows gone for every recipient; badges drop
10. Maya opens her panel → empty state; Owner attempts Maya's old notification id directly → 404
```

**Negative E2E checks (cheap):**

- **Cross-recipient leak:** Maya's notification id under Owner's session → 404 byte-equal to random cuid.
- **Sourceless mint:** `POST /notifications` directly → 404 (no create route).
- **Archived tolerance:** archive the issue → rows still listed, badge still counts, click lands read-only; archive the workspace → panel still fully usable.
- **Clear-all caution:** clear-all without `confirm:true` → 400, inbox intact; with it → empty + badge 0.

Scope discipline: journey + negatives are the mandatory F6 E2E suite; exhaustive cases stay in 10.1–10.2. Cycle-silence (no rows for cycle events) is asserted at the integration layer, not E2E.

---

## 11. Cross-cutting concerns

| Concern | Approach |
|---|---|
| **Rate limiting** | Poll-guard on #2 (generous burst, 60/min per user — polling must never 429 under normal tabs; wiring finalized at F12). Mutations inherit the global limiter; no per-row create limit exists (no create route). Emission fan-out is bounded by source-route limits (issues/comments create caps). |
| **Polling discipline** | `useUnreadCount`: ~60s interval, paused when tab hidden, jittered to avoid thundering herds; panel queries fetch on open/focus only (arch §11 — no WebSockets, no push). Badge is eventually consistent by design; tests use a shortened interval, never real waiting. |
| **Sorting / ordering** | Server is the only orderer: `(createdAt DESC, id DESC)` always. Client appends pages verbatim. No "unread pinned first" second ordering in MVP (spec §3.2: newest-first + filter). |
| **Pagination** | Cursor only (both panel and unread-subset walks). Default `limit=25`, max 100. `nextCursor: null` is the end signal. Count (#2) is never paginated. |
| **Audit** | No audit table — `createdAt` + first-`readAt` are the audit surface. Emission provenance is reconstructible from the source rows (issue history / comment joins). |
| **Copy ownership** | Client-side presenter from card facts (spec Q1). No localized/server sentences to version; no copy drift between panel and tests (component tests assert the presenter directly). |
| **Search** | No `q` param here or ever — F10 excludes this table (§7 of data-model). Panel filtering is `unreadOnly` + `workspaceId` only. |
| **Realtime** | Explicitly none in MVP (spec §6, arch §11). The worker slot stays reserved; push/email are the post-MVP paths, each requiring a new outbox/delivery design, not an extension of this one. |

---

## 12. References

- Shipyard: `features/notifications/spec.md`, `features/notifications/data-model.md`, `features/workspace/api-design.md` (envelope, leak-free 404 precedent), `features/members/api-design.md` (`confirm: true` precedent), `features/issues/api-design.md` (cursor pagination, assignment call-site), `features/comments/api-design.md` (mention call-site, retraction contract, permalink convention), `features/auth/api-design.md` (session), `00-architecture.md` §5–§8/§11, `ADR-001`–`ADR-003`, `Implementation Plan.md` F6
- Prisma: referential actions, indexes — `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- Cursor pagination: `https://www.prisma.io/docs/orm/prisma-client/queries/pagination#cursor-based-pagination`
- PostgreSQL partial indexes: `https://www.postgresql.org/docs/current/indexes-partial.html`

---

*Next artifact: implementation (plan §5 Steps 3–7) — Prisma migration (`notification` + `NotificationType` + back-relations + `notification_unread_idx`) → module code (routes/controller/service/repository + shared schemas + `createAssignment`/`createMention`/`deleteForComment` internals) → F5/F8 call-site wiring (issues reassign tx, comment create/delete txs) → web slice (bell + badge poll, panel, deep-link landing, copy presenter) → tests → `pnpm check`. No further design doc needed; data-model + api-design cover F6's technical design.*
