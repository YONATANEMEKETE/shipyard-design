# Comments — API Design

**Status:** Draft for review
**Last updated:** 2026-09-04
**Sources:** `features/comments/spec.md` · `features/comments/data-model.md` (locked — `comment` + `comment_mention`, D1–D10) · `features/workspace/api-design.md` (F2 precedent — `:slug` context, guard chain, archived matrix, error envelope) · `features/members/api-design.md` (F3 precedent — RBAC pipeline, `confirm: true` convention, directory as suggestion source) · `features/issues/api-design.md` (F5 precedent — issue-scoped nesting, cursor pagination, `ISSUE_ARCHIVED` pattern) · `features/notifications/spec.md` (F6 consumer — mention emission, atomicity) · `features/auth/api-design.md` (F1 — Better Auth session) · `00-architecture.md` §5–§8 · `ADR-001`–`ADR-003` · `Implementation Plan.md` F8

> **Principle:** identical to issues (F5) — every route is hand-written Shipyard code through the canonical pipeline:
>
> ```text
> route → validation → permission check → controller → service → repository → Prisma
> ```
>
> Better Auth handles identity; this module owns **authorization** for comment data — who may read/post (any member) and who may mutate a row (its author only — roles never override, spec rule 3). No new auth primitive; it reuses the F2/F3 guard chain verbatim plus an authorship service check.

---

## 1. Base path & conventions

| Concern | Choice |
|---|---|
| Base path | `/api/v1/workspaces/:slug/issues/:issueId/comments` and `/api/v1/workspaces/:slug/issues/:issueId/comments/:commentId` — nests under the issue like labels/history nest under issues in F5; `:slug` is the F2 immutable workspace token; `:issueId`/`:commentId` are `cuid()` row ids (never display text — same reasoning as projects D5). |
| Next.js proxy | Browser never hits the API directly (ADR-003); `apps/web` forwards `/api/v1/*` → `http://api:4000/api/v1/*`, cookies forwarded. |
| Auth transport | HttpOnly Better Auth session cookie read by `requireSession` (F1) — `req.session.userId` is the only identity input. |
| Validation | Zod schemas from `packages/shared` (`data-model.md` §4) at the route boundary. Content bounds 1–10,000 chars trimmed (D5); cursor opaque base64url of `(createdAt, id)`. |
| Envelope | Success: resource JSON directly (or `{ comments: [...], nextCursor }` for collections). Failure: `{ "error": { "code", "message", "details"? } }` via the global error handler. |
| Workspace context | Reuses F2 `resolveWorkspaceContext(:slug)` verbatim — one authoritative resolution per request, leak-free `404 WORKSPACE_NOT_FOUND`. |
| Archived enforcement (workspace) | Mutating routes use `resolveWorkspaceContext({ rejectArchived: true })`; `GET` routes pass `rejectArchived: false`. |
| Issue/comment-level read-only | Enforced in the **service**: archived issue freezes all comment writes (§6.2, D9); authorship gates edit/delete (§6.3) — restore/delete of the issue stay in the issues module. |
| Mention suggestions | No new endpoint — the composer reads `GET /api/v1/workspaces/:slug/members` (F3 #1, any member) for current member names and filters client-side. |

---

## 2. Endpoint inventory

Five endpoints cover every behavior in `spec.md` §2–§5 and `data-model.md` §6. Mention parsing, notification fan-out, and suggestion data are **not** endpoints (§5.2, §11). No extras.

### 2.1 Workspace-scoped — comments (all nested under the parent issue)

All under `/api/v1/workspaces/:slug/issues/:issueId/comments...`, all through the §4 guard chain.

| # | Method | Path | Guard chain | Spec behavior |
|---|---|---|---|---|
| 1 | `GET` | `/api/v1/workspaces/:slug/issues/:issueId/comments` | `requireSession` → member (any role) | §2/§3.1 List conversation chronologically (oldest first). Query: cursor `cursor` + `limit` (default 50, max 100). Allowed on archived issues (existing comments remain visible). Cards carry `mentions[]` inline. |
| 2 | `GET` | `/api/v1/workspaces/:slug/issues/:issueId/comments/:commentId` | `requireSession` → member (any) | §2 Detail — one card (notification deep-link target, permalink). Allowed on archived issues. Scoped: `:commentId` validated against `:issueId` + `:slug`. |
| 3 | `POST` | `/api/v1/workspaces/:slug/issues/:issueId/comments` | `requireSession` → member (any) + `rejectArchived` | §3.1/§3.3/§4.1 Create — any member. Body `createCommentSchema` (`{ content }`; mentions derived server-side, never trusted from the client). Rejected on archived issues (§6.2). Mentioned members notified in the same tx. |
| 4 | `PATCH` | `/api/v1/workspaces/:slug/issues/:issueId/comments/:commentId` | `requireSession` → member (any) + `rejectArchived` | §3.2/§4.2 Edit own comment — full content replacement, sets `editedAt`. Body `updateCommentSchema`. Author-only (service, §6.3 — Owner/Admin included in the rejection). Joins silently recomputed, zero re-notify. Rejected on archived issues. |
| 5 | `DELETE` | `/api/v1/workspaces/:slug/issues/:issueId/comments/:commentId` | `requireSession` → member (any) + `rejectArchived` | §4.3 Delete own comment — confirmed `{ confirm: true }`. Author-only. Cascades joins + its mention notifications in the same tx (no dead links, D8). Rejected on archived issues. |

> **Why nested under the issue:** a comment has no meaning outside its issue (spec rule 1) and every permission question about it ("is the parent archived? is the issue in my workspace?") needs the issue row first. Flat `/comments/:id` would re-resolve the issue anyway — nesting makes the scope explicit in the URL, same reasoning as F5's `/issues/:issueId/labels` and `/issues/:issueId/history`.
>
> **Why any member can write:** comments are conversation, not privileged lifecycle — same divergence as issues (F5 rule 10), opposite of cycles/projects. The restriction that matters here is authorship (#4/#5), not role.
>
> **Why no archive/restore endpoints:** comments have no archive dimension of their own (data-model D9) — freezing comes from the parent issue, and deletion is the only exit.

---

## 3. Context resolution

### 3.1 Workspace + issue + comment lookups — chained, each scoped

```text
req.params.slug ──resolveWorkspaceContext──▶ { workspaceId, role, ... }   (404 WORKSPACE_NOT_FOUND generic)
  │
req.params.issueId ──findFirst(where: { id, workspaceId })──▶ issue       (404 ISSUE_NOT_FOUND scoped)
  │
req.params.commentId ──findFirst(where: { id, issueId, workspaceId })──▶ comment
                                                                     (404 COMMENT_NOT_FOUND scoped)
```

- No workspace with slug **or** no membership ⇒ generic `404 WORKSPACE_NOT_FOUND` (no existence leak, F2).
- Issue id from another workspace ⇒ `404 ISSUE_NOT_FOUND` (indistinguishable from nonexistent — F5 convention).
- Comment id from another issue or workspace ⇒ `404 COMMENT_NOT_FOUND` — triple-scoped (`id` + `issueId` + `workspaceId`) so a comment id smuggled under a sibling issue's URL never resolves. Defense in depth: service reasserts `comment.issueId === issue.id`.
- The issue row carries `archivedAt` — service uses it for the freeze-all matrix (§6.2). The comment row carries `authorId` — service uses it for the authorship matrix (§6.3). Neither is ever resolved cross-workspace.
- List (#1) and create (#3) resolve workspace + issue only (no `:commentId`); #2/#4/#5 resolve all three.

### 3.2 Mention resolution (write-time, data-model D6)

Owned by the comments service inside the create/edit transaction — not a route: parse `content` with `mentionTokenRegex`, match case-insensitively against current workspace members (full `user.name` or any whitespace-separated word), dedup in encounter order, write `comment_mention` rows for hits only. Membership snapshot is read in-tx so concurrent leave during post cannot notify a leaver. Unknown tokens produce no rows and no errors.

---

## 4. Guard chain (canonical, mirrors issues §4)

### 4.1 Comment routes (#1–#5)

```text
requireSession                     ← F1: valid Better Auth session else 401
  │
resolveWorkspaceContext(slug)      ← F2 shared middleware
  │                                  404 generic on miss/non-membership (no leak)
  │                                  409 WORKSPACE_ARCHIVED when rejectArchived && ARCHIVED (#3–#5)
  │
issueById :issueId                 ← module lookup scoped to workspaceId → 404 ISSUE_NOT_FOUND
  │
commentById :commentId             ← (#2/#4/#5) triple-scoped lookup → 404 COMMENT_NOT_FOUND
  │
service preconditions              ← issue-archived freeze (§6.2): archived ⇒ 409 ISSUE_ARCHIVED (#3–#5)
                                     authorship (§6.3): authorId !== caller ⇒ 403 NOT_COMMENT_AUTHOR (#4–#5)
                                     content bounds + confirm:true — inside the same transaction as writes
```

Deliberate divergences, stated once: **no `requireWorkspaceRole` anywhere** in this module (all five routes accept any member — the write privilege is membership, the mutation privilege is authorship); **no role override** on #4/#5 (Owner/Admin editing another author's comment get the same `403` as a Member); workspace-archived freeze at the guard layer, issue-archived freeze + authorship reasserted in the service.

Rules reaffirmed (F2–F7): URL carries all scope (workspace + issue + comment), no hidden server state; membership resolved once; checks in named guards or service preconditions, never ad-hoc controller queries.

---

## 5. Request/response contracts

Schemas from `packages/shared` (`data-model.md` §4). Route handlers validate bodies, params, and query before anything else.

### 5.1 Comments

| Endpoint | Body / Query | Success response |
|---|---|---|
| #1 list | query `?limit=1..100` (default `50`) · `?cursor=<opaque>` | `200` + `{ comments: commentCardSchema[], nextCursor: string \| null }` chronological (oldest first); `nextCursor null` ⇒ end. Cards carry `mentions[]` inline (no second fetch). |
| #2 detail | — | `200` + `commentCardSchema` |
| #3 create | `createCommentSchema` `{ content (required, trim 1–10k) }` — no `mentions` field; client-sent handles (if any) are ignored, server re-parses authoritatively | `201` + `commentCardSchema` (mentions resolved inline, `editedAt: null`) |
| #4 update | `updateCommentSchema` `{ content (required, full replacement) }` | `200` + `commentCardSchema` (`editedAt` set, joins recomputed silently) |
| #5 delete | body `{ confirm: true }` | `200` + `{ deletedCommentId }` — joins + its mention notifications removed atomically |

Validation & precondition details:

- `#3`/`#4`: `content` trimmed by Zod; empty after trim or >10,000 chars ⇒ `400 VALIDATION_ERROR`. No minimum-word or mention-required rule — plain comments without mentions are the common case.
- `#3`: issue archived ⇒ `409 ISSUE_ARCHIVED` (freeze-all, D9). Unknown `@tokens` ⇒ still `201` (literal text, no rows — spec rule 8, never a 400). Self-mention ⇒ join row written, no notification emitted (same-discipline as F5 same-person no-op).
- `#4`: non-author (including Owner/Admin) ⇒ `403 NOT_COMMENT_AUTHOR`. Same-content body ⇒ no-op: no write, `editedAt` untouched, `200` with the unchanged card (mirrors F5 no-op discipline). Issue archived ⇒ `409 ISSUE_ARCHIVED`. Edit re-parses mentions vs *current* members; added handles emit nothing (rule 4 — edits never re-notify).
- `#5`: non-author ⇒ `403 NOT_COMMENT_AUTHOR` (checked before confirmation semantics). Missing literal `confirm: true` ⇒ `400 CONFIRMATION_REQUIRED` (same precedent as labels/cycles delete). Issue archived ⇒ `409 ISSUE_ARCHIVED`. Response carries only the id — the row is gone (no tombstone); clients remove it from the cached list.
- `#1`: unknown/malformed cursor ⇒ `400 VALIDATION_ERROR`. Cursor is bound to `(createdAt, id)` ASC — the only order this endpoint serves, so no filter-change invalidation problem (unlike issue-list sort-specific cursors). `limit` out of range ⇒ `400 VALIDATION_ERROR`.
- `#2`/`#4`/`#5`: `:commentId` not under `:issueId` in this workspace ⇒ `404 COMMENT_NOT_FOUND` (triple-scoped, §3.1) — never a cross-issue leak.

### 5.2 Non-endpoints (explicitly not routes)

| Concern | Served by | Notes |
|---|---|---|
| Mention suggestions (composer `@` autocomplete) | `GET /api/v1/workspaces/:slug/members` (F3 #1, any member) | Client filters by typed prefix; server adds nothing. Current-members-only falls out of the directory. |
| Mention notifications (list/read/delete) | Notifications module (F6) | This module only emits (create) and retracts (delete) via the §7 contract; notification reads never touch comment routes. |
| Issue delete cascade | Issues module `DELETE #7` | Calls down through `comment.issueId Cascade` + notification cleanup in the same tx (§8.5). No comment-route involvement. |

---

## 6. Read-only / authorship enforcement matrices

### 6.1 Workspace-level (`workspace.status = ARCHIVED`)

| Endpoint | While ARCHIVED | Rationale |
|---|---|---|
| #1 list, #2 detail | ✅ allowed | Read-only — the frozen workspace stays browsable, conversations intact |
| #3–#5 all writes | ❌ `409 WORKSPACE_ARCHIVED` | No conversation edits in a frozen container |

Enforced at the guard layer (`rejectArchived: true`) for #3–#5.

### 6.2 Issue-level (`issue.archivedAt` set — own lifecycle, active workspace)

Archived **issues** freeze their whole conversation (D9) while the **workspace** is active — two independent axes. Enforced in the service:

| Endpoint | While issue archived | Notes |
|---|---|---|
| #1 list, #2 detail | ✅ allowed | Existing comments remain visible (spec §3.1) |
| #3 create | ❌ `409 ISSUE_ARCHIVED` | Cannot receive new comments |
| #4 update | ❌ `409 ISSUE_ARCHIVED` | Frozen record — restore the issue first (checked before authorship so archived+foreign returns the freeze code, not the authorship code) |
| #5 delete | ❌ `409 ISSUE_ARCHIVED` | Same — even the author's own delete waits for restore |

### 6.3 Authorship-level (`comment.authorId` vs caller — active issue, active workspace)

Roles never override (spec rule 3) — enforced in the service on #4/#5:

| Caller | #4 update / #5 delete | Notes |
|---|---|---|
| Author (any role, incl. Member) | ✅ allowed | The only writer |
| Non-author Member | ❌ `403 NOT_COMMENT_AUTHOR` | |
| Non-author Admin / Owner | ❌ `403 NOT_COMMENT_AUTHOR` | No moderation power in MVP — same code, no privilege hint |
| Unauthenticated | ❌ `401 UNAUTHENTICATED` | Guard layer, before authorship |

Defense in depth: service reasserts `authorId` + `archivedAt` even though guards already ran. Check order inside mutations: workspace-archived (guard) → issue-archived → authorship → confirmation/content — so each failure returns its most specific code.

---

## 7. Error codes (Comments module)

Global error handler converts typed domain errors; controllers never build envelopes by hand.

| Code | HTTP | When | Notes |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | Zod body/param/query failure (empty/overlong content, bad `limit`/`cursor`, bad cuid params) | `details` lists field paths |
| `CONFIRMATION_REQUIRED` | 400 | #5 without literal `confirm: true` | Same precedent as labels/cycles |
| `WORKSPACE_NOT_FOUND` | 404 | Unknown `:slug` or caller not a member — deliberately identical | No existence leak (§3.1) |
| `ISSUE_NOT_FOUND` | 404 | `:issueId` not in this workspace | Scoped (F5 convention) |
| `COMMENT_NOT_FOUND` | 404 | `:commentId` not under `:issueId` in this workspace | Triple-scoped (§3.1) |
| `NOT_COMMENT_AUTHOR` | 403 | #4/#5 caller is not the comment's author (any role, incl. Owner/Admin) | Authorship, not role — no `FORBIDDEN_ROLE` in this module |
| `ISSUE_ARCHIVED` | 409 | #3/#4/#5 while the parent issue is archived | Freeze-all (D9); reads unaffected |
| `WORKSPACE_ARCHIVED` | 409 | Mutating op while the workspace is `ARCHIVED` (§6.1) | Restorable via workspace restore |
| `UNAUTHENTICATED` | 401 | Missing/expired session cookie | F1 `requireSession` |
| `RATE_LIMITED` | 429 | Per-route create/update limits (wiring finalized at F12; global limiter exists) | `Retry-After` header |

No `FORBIDDEN_ROLE` (every route accepts any member), no `ALREADY_ARCHIVED`/`NOT_ARCHIVED` (comments have no archive dimension), no mention-not-found code (unknown handles are literal text, never errors).

---

## 8. Sequences

### 8.1 Create with mentions (spec §4.1)

```text
Member → POST /api/v1/workspaces/:slug/issues/:issueId/comments {content:"@maya can you check this?"}
→ requireSession ✓ → resolveWorkspaceContext ✓ (any member) → issueById ✓ → Zod validate
→ service tx {
     assert issue.archivedAt IS NULL else 409 ISSUE_ARCHIVED
     INSERT comment { workspaceId, issueId, authorId: caller, content: trimmed }
     mentions = parse(content) vs current members in-tx → dedup encounter order
       → "@maya" hits Maya Chen ⇒ [maya]; "@ghost" ⇒ no row (literal)
     INSERT comment_mention rows
     // F6 hook per distinct hit except self: notificationsService.createMention({...}, tx) (§7)
   } → 201 card (mentions: [{Maya}])
→ Maya's unread badge increments on next ~60s poll; Bob (unmentioned) sees nothing
→ concurrent member-leave mid-post: leave tx commits first ⇒ leaver no longer current ⇒ literal, no notify
```

### 8.2 Edit own comment — silent recompute (spec §4.2)

```text
Author → PATCH .../comments/:commentId {content:"@carol actually you — @maya nvm"}
→ guard ✓ → service tx {
     assert issue not archived else 409; assert authorId === caller else 403 NOT_COMMENT_AUTHOR
     same content ⇒ no-op 200 (editedAt untouched)
     else UPDATE content + editedAt=now()
     recompute joins vs current members: Maya stays, Carol added
     // zero notification writes — Carol gets nothing (rule 4)
   } → 200 card (mentions: [Maya, Carol], editedAt set, "(edited)" renders)
```

### 8.3 Delete own comment — retraction (spec §4.3)

```text
Author → DELETE .../comments/:commentId {confirm:true}
→ guard ✓ → service tx {
     assert issue not archived else 409; assert authorship else 403 (before confirm check)
     assert confirm:true else 400 CONFIRMATION_REQUIRED
     DELETE comment → cascades comment_mention joins
     delete mention notifications for commentId (D8 — FK Cascade or deleteForComment in-tx)
   } → 200 { deletedCommentId } → client drops the row from the cached conversation
→ Maya's already-delivered mention notification for this comment is gone (no dead link)
→ assignment notifications + sibling comments + issue untouched
```

### 8.4 Non-author attempt (spec rule 3)

```text
Owner (not author) → PATCH .../comments/:commentId {content:"fixed wording"}
→ guard ✓ (any member passes) → service: authorId !== caller → 403 NOT_COMMENT_AUTHOR
→ Owner sees the same code a Member would — UI hides the buttons, API enforces it
```

### 8.5 Issue delete cascade (F5 ↔ F8, closes the F5 leg)

```text
Owner/Admin → DELETE .../issues/:issueId (F5 #7) → issues service tx {
  DELETE comment WHERE issueId=:id   // F8 leg, now live → cascades comment_mention joins
  delete mention notifications for those commentIds (D8 chain)
  DELETE issue (cascades issue_label + issue_history + F6 assignment notifications per F5 §6.5)
} → comments and their notifications vanish with the issue (issues rule 9); users/members untouched
```

### 8.6 List walk (chronological cursor)

```text
GET .../comments?limit=50 → { comments: [...50 oldest], nextCursor: "ey..." }
GET .../comments?cursor=ey...&limit=50 → next 50 (older → newer, ASC)
→ nextCursor null ⇒ end reached; new comments appear on refetch (no realtime in MVP)
→ archived issue: same walk, 200 (reads frozen-open)
```

---

## 9. Module layout

### 9.1 API — `apps/api/src/features/comments/`

```text
features/comments/
├── routes.ts        # router: nested path defs → middleware chain → controller; Zod validated at entry
│                    # (#1–#5; issue lookup middleware shared with issues module guards)
├── schemas.ts       # route-local param/query coercion (slug/issueId/commentId params,
│                    # limit/cursor query, confirm:true body); shared schemas in packages/shared
├── controller.ts    # HTTP concerns only: parse req/query, call service, map result/errors
├── service.ts       # business rules: issue-scope checks, archived freeze, authorship checks,
│                    # mention parse/resolve/dedup, create/edit/delete txs, notification-contract calls
├── repository.ts    # Prisma access only (comment + mention-join reads/writes, chronological page query)
└── errors.ts        # typed domain errors → global handler maps to §7
```

Shared guards reused (not owned by this module):

```text
common/guards/
├── require-session.ts           # (F1)
└── workspace-context.ts         # (F2) resolveWorkspaceContext(:slug)
```

No `require-workspace-role.ts` usage — this module has no role-gated route. Cross-module contracts owned elsewhere, called here:

```text
membersService  → resolveWorkspaceContext, current-member snapshot (mention resolution, §3.2)
issuesService   → issueById lookup shape (archivedAt for the freeze check)
notificationsService (F6) → createMention(event) on #3, deleteForComment(commentId) on #5, in-tx (§7)
```

### 9.2 Shared — `packages/shared/src/comments/`

Re-exports from `data-model.md` §4 — the canonical place:

- Bounds/rules: `commentContentSchema`, `mentionTokenRegex`
- Request: `createCommentSchema`, `updateCommentSchema`, `commentListQuerySchema`
- Response: `commentAuthorCardSchema`, `commentMentionCardSchema`, `commentCardSchema`, `commentListPageSchema`, delete-result schema (`{ deletedCommentId }`)

### 9.3 Web — `apps/web`

| Surface | Route | Reads/Writes |
|---|---|---|
| Issue detail conversation | `/w/:slug/issues/:issueId` (comments section) | #1 list (cursor walk / "load more"), #3 create (composer), live `mentions[]` link rendering |
| Composer + autocomplete | over conversation | Member directory (F3 list) filtered by typed `@` prefix; submit sends `POST #3` with raw content only |
| Comment row | in conversation | #2 detail (permalink/deep-link from notification), #4 inline edit (author only), #5 delete confirm (author only), `(edited)` + time from `editedAt` |
| Empty states | over conversation | No comments yet; load-more end; archived-issue composer disabled ("restore to comment") |

Data access via TanStack Query hooks (like issues): standard (non-infinite) query for the first page + `fetchMore` on `nextCursor` for older→newer growth, mutations pessimistic for create/edit/delete (authoritative — no optimistic mention fan-out). Mention links render from `mentions[]` (server-resolved userIds), never client-parsed.

All surfaces ship with loading, error, empty (no comments), and permission-aware states (edit/delete hidden for non-authors of each row — per-row, not per-role; composer hidden for non-members, which the guard makes unreachable anyway; archived-issue banner with restore path).

---

## 10. Testing strategy

Three layers (mirrors issues §10). Tooling provisioned by F1/F2; no new deps.

### 10.1 API integration tests

Supertest against `createApp()`, real Postgres via Testcontainers + migrations. Seeded helpers: `createVerifiedUser`, `createWorkspaceAs(owner)`, `addMember(workspace, user, role)`, `createIssue(workspace, overrides)`, `createComment(issue, author, overrides)`.

| Case | Covered by |
|---|---|
| Happy paths ×5 endpoints | Supertest suite per group (comments list/detail/mutations) |
| Invalid input (empty/whitespace-only, overlong >10k, bad cuid params, bad `limit`/`cursor`) | `400 VALIDATION_ERROR` |
| Missing `confirm: true` (#5) | `400 CONFIRMATION_REQUIRED` |
| Unauthenticated ×5 | `401 UNAUTHENTICATED` |
| Non-member access (real slug, foreign user) | `404 WORKSPACE_NOT_FOUND` — assert byte-equal to unknown-slug (leak test) |
| Any member (incl. Member role) creates/lists/reads | `201`/`200` (no role gate — assert Member succeeds where projects/cycles would 403) |
| Edit/delete by non-author Member | `403 NOT_COMMENT_AUTHOR` |
| Edit/delete by non-author Owner/Admin | `403 NOT_COMMENT_AUTHOR` (no override — assert Owner gets the same code) |
| Edit by author → `editedAt` set, content replaced | `200` + DB assertions |
| Same-content edit → no-op (`editedAt` untouched) | `200` + `editedAt` null still + no extra writes |
| Create on archived issue | `409 ISSUE_ARCHIVED` |
| Edit/delete on archived issue (even by author) | `409 ISSUE_ARCHIVED` (freeze-all; assert before-authorship ordering with archived+foreign case) |
| List/detail on archived issue | `200` (frozen-open reads) |
| Cross-issue — comment id under sibling issue URL | `404 COMMENT_NOT_FOUND` (triple-scope) |
| Cross-workspace — issue/comment ids from another workspace | `404 ISSUE_NOT_FOUND` / `404 COMMENT_NOT_FOUND` scoped |
| Mentions — `@known` resolves row + notifies; `@unknown` literal, `201`, no row, no notify | `201` + join-count + notification-count assertions |
| Mentions — duplicate `@maya @maya` collapses to one row + one notification | join-count 1 + notify-count 1 |
| Mentions — case-insensitive (`@MAYA` hits `Maya`) | resolve + notify |
| Mentions — word-match (`@maya` hits `Maya Chen`) | resolve + notify |
| Mentions — ambiguous (two Mayas) notifies both | notify-count 2, joins 2 |
| Mentions — member who left stays literal, no notify | `201`, zero joins for that token |
| Mentions — self-mention writes join, emits nothing | join-count 1, notify-count 0 |
| Mentions — edit adding a handle emits nothing, joins recomputed | notify-count unchanged, joins match v2 text |
| Mentions — edit removing all handles clears joins, old notifications stay | joins 0, prior notify rows intact |
| Delete — joins + its mention notifications gone, siblings/issues/users alive | `200` + join/notify-count assertions |
| Delete rollback on notification-cleanup failure | comment still present |
| Issue delete (F5 #7) → its comments + joins + mention notifications gone | cross-module test (calls issues route, asserts comments) |
| Archived workspace writes (#3–#5) | `409 WORKSPACE_ARCHIVED` |
| Archived workspace reads (#1, #2) | `200` |
| List — chronological ASC order, cursor walk (page 1 → 2 → end `nextCursor: null`), invalid-cursor 400 | ordering + `nextCursor` assertions |
| Mention suggestions — no endpoint (assert composer uses members list; no `GET .../mentions` route exists) | route-table assertion / 404 on guessed path |

### 10.2 Component tests (web) — MSW mocks of `/api/v1/workspaces/:slug/issues/:issueId/comments*` + `.../members`

| Surface | Cases |
|---|---|
| Conversation | Renders `commentCardSchema` list oldest-first; empty state (no comments); "load more" walks `nextCursor`; end state hides the button |
| Comment row | Shows author, time, content, `mentions[]` as member links (from server ids, not client parse); `(edited)` + time only when `editedAt` set |
| Per-row authorship | Edit/delete buttons visible only on own rows (assert per-row, not per-role — Owner sees them only on own rows); others' rows show no affordances |
| Composer | Sends `POST #3` with `{ content }` only (MSW spy asserts no client-computed mentions); empty submit blocked client-side *and* `400` inline; `@` autocomplete filters mocked members list; unknown handle submits fine as literal |
| Inline edit | Author edits → sends `PATCH #4`; `NOT_COMMENT_AUTHOR` (stale row, authorship changed) shows retry; success shows `(edited)` immediately |
| Delete dialog | Author confirms → sends `DELETE #5` with `confirm:true`; missing confirm blocked; success drops the row; `ISSUE_ARCHIVED` shows restore path |
| Archived-issue banner | Composer disabled + edit/delete hidden with "restore to comment" messaging (writes would 409) |
| Notification deep-link | Clicking a mention notification routes to issue detail scrolled to `#comment-:id` (`GET #2` fills the permalink when the row is off-page) |
| Error envelope rendering | Every surface renders MSW-served `{error:{code,message}}` as friendly states, never raw dumps |
| Archived workspace wrapper | All mutating affordances disabled with frozen messaging |

Rules: components never re-implement business rules (e.g., "Owner can edit anyone" is false and API-enforced; web just hides per-row controls). Tests assert wire behavior + rendered state.

### 10.3 End-to-end journey — golden path

Playwright against the composed stack (web + api + Postgres, reset between runs).

**Journey — comment lifecycle with mentions (core)**

```text
1. Owner + Member (Maya) exist in a workspace with an issue (F3/F5)
2. Member posts "@owner please look" → comment appears; Owner's badge increments on next poll
3. Owner opens notification → lands on issue detail at the comment (permalink)
4. Member edits own comment to add more text → "(edited)" appears; no second notification fires
5. Owner attempts to edit Member's comment (crafted request) → 403; button was never rendered
6. Member posts "@ghost hello" (unknown) → 201 as literal text, no notification anywhere
7. Member deletes own comment (confirm) → row gone; its mention notification gone with it
8. Owner archives the issue → composer disabled; posting directly → 409; conversation still readable
9. Owner restores the issue → commenting works again
```

**Negative E2E checks (cheap):**

- **Cross-issue leak:** second issue's comment id under first issue's URL → 404.
- **Role override attempt:** Owner sends `PATCH`/`DELETE` on Member's comment directly → 403 `NOT_COMMENT_AUTHOR`.
- **Archived freeze:** archived issue + direct `POST`/`PATCH`/`DELETE` → 409 `ISSUE_ARCHIVED`; restore → writes succeed.
- **Archived workspace freeze:** archive the workspace, then attempt post/edit/delete → 409; conversation still readable.

Scope discipline: journey + negatives are the mandatory F8 E2E suite; exhaustive cases stay in 10.1–10.2. Dashboard-recent-activity and search-over-comments E2E land with F9/F10.

---

## 11. Cross-cutting concerns

| Concern | Approach |
|---|---|
| **Rate limiting** | Per-route create (30/min per user per issue — conversation pace, mirrors issues create), update (20/min), delete (10/min). Memory for MVP; global limiter exists; wiring finalized at F12. Caps double as mention fan-out abuse control. |
| **Validation encoding** | Content is plain `Text` end-to-end (basic formatting rendered client-side from the same string; no server markdown pipeline in MVP). Cursor is opaque base64url of `(createdAt, id)` — clients never parse it. |
| **Sorting / ordering** | Server is the only orderer: `(createdAt ASC, id ASC)` always. Client appends pages verbatim; no client-side re-sort, no board grouping (unlike issues/cycles list-vs-kanban duality — a conversation has one order). |
| **Pagination** | Cursor only (spec conversation reads). Default `limit=50`, max 100. No offset mode. `nextCursor: null` is the end signal; live growth is refetch, not realtime (arch §11). |
| **Audit** | No comment-history table in F8 (spec rule 6). `editedAt` + `createdAt`/`updatedAt` are the audit surface; per-issue `CYCLE_CHANGED`-style event rows are not emitted for comments. |
| **Notifications hook** | Comments exposes the mention event inside its tx (§8.1); F6 owns the `notification` rows. Edit emits nothing; delete retracts (§8.3). No comment notifications beyond mentions in MVP. |
| **Search** | No `q` param in F8; F10 adds generated `tsvector` on `comment(content)` + GIN + grouped endpoint without changing these contracts. |

---

## 12. References

- Shipyard: `features/comments/spec.md`, `features/comments/data-model.md`, `features/workspace/api-design.md` (guard chain, archived matrix, envelope), `features/members/api-design.md` (RBAC pipeline, `confirm: true`, directory-as-suggestions), `features/issues/api-design.md` (issue nesting, cursor pagination, `ISSUE_ARCHIVED`), `features/notifications/spec.md` (mention emission §3.1, atomicity rule 7, cascade rule 5), `features/auth/api-design.md` (session), `00-architecture.md` §5–§8, `ADR-001`–`ADR-003`, `Implementation Plan.md` F8
- Prisma: referential actions, composite ids — `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- Cursor pagination: `https://www.prisma.io/docs/orm/prisma-client/queries/pagination#cursor-based-pagination`

---

*Next artifact: implementation (plan §5 Steps 3–7) — Prisma migration (`comment` + `comment_mention` + back-relations, no raw SQL) → module code (routes/controller/service/repository + shared schemas + mention parse/resolve + notification-contract calls) → web slice (conversation, composer + autocomplete, inline edit, delete confirm, edited indicator, archived banners) → tests → `pnpm check`. Dashboard recency and full-text search land with F9/F10 via §7/§11.*
