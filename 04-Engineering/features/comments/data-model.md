# Comments — Data Model

**Status:** Draft for review
**Last updated:** 2026-09-04
**Sources:** `features/comments/spec.md` · `features/issues/data-model.md` (F5 precedent — `issue` cascade contract, `issue_label` join pattern, history actor convention, cursor pagination) · `features/notifications/spec.md` (F6 consumer — mention event, atomicity rule 7, no-dead-links) · `features/members/data-model.md` (F3 precedent — membership vs user rows, cascade convention) · `features/workspace/data-model.md` (F2 precedent — identity, cascade) · `features/auth/data-model.md` (identity — `user.name` is the display name) · `00-architecture.md` §5, §8, §9 · `ADR-001` (Prisma + Postgres) · `ADR-002` (shared contracts) · `Implementation Plan.md` F8
**Owner:** `apps/api` — Prisma-owned (hand-modeled, like workspace/members/projects/issues/cycles).

---

## 1. Overview

Comments owns the **discussion layer**: messages attached to issues with author-only mutation, chronological reads, and `@mention` resolution that feeds Notifications.

Two new tables:

| Table / Change | Purpose | Formalized by |
|---|---|---|
| `comment` | The message: identity, scope, parent issue, author, content, edited marker | **F8 (this milestone)** |
| `comment_mention` | Join: resolved mentions per comment (distinct user per comment, the notify list) | **F8** |

`issue` is owned by F5 — F8 wires the descendant leg (`comment.issueId → issue`, `onDelete: Cascade`) that F5's delete transaction already anticipates (issues `data-model.md` §6.5/§7: "comments die with the issue"). `notification` rows for mentions are owned by F6 — F8 calls the mention-event contract inside the comment-create transaction and the delete contract inside the comment-delete transaction (§6.1/§6.3, §7).

`comment` has no `archivedAt`, no status, no soft-delete flag: lifecycle is inherited from the parent issue (archived issue ⇒ frozen, §6.4). Deleted comments are physically removed with their mention joins — no tombstones (spec §3.1).

---

## 2. Core schema (Prisma-owned)

### 2.1 `comment`

One row per message. Immutable internal identity via `cuid()`; no display identifier (comments are addressed by position in the issue conversation, never by URL slug — detail reads go through the issue).

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK `@default(cuid())` | Immutable internal identifier. Never a URL segment alone; reads are always `WHERE issueId` (§6.5). |
| `workspaceId` | `String` | FK → `workspace.id` `onDelete: Cascade` + `@@index([workspaceId])` | Denormalized scope (same convention as `issue_history.workspaceId`): workspace delete cleans comments even mid-tx; powers workspace activity queries without joining `issue`. |
| `issueId` | `String` | FK → `issue.id` `onDelete: Cascade` + `@@index([issueId])` | Exactly one issue (spec rule 1). Issue delete removes its comments (issues rule 9). |
| `authorId` | `String` | FK → `user.id` `onDelete: Restrict` + `@@index([authorId])` | Exactly one author (rule 1). `Restrict` — user-row deletion is blocked while comments exist; authorship is structural, not service-only (D3). Leaving a workspace does **not** touch this (membership ≠ user row — leavers' comments survive). |
| `content` | `String` | `@db.Text` | Required, trimmed server-side. Bounds 1–10,000 chars enforced in Zod (D5); `Text` column unbounded by design, bound lives at the edge like `issue.description`. |
| `editedAt` | `DateTime?` | — | `null` ⇒ never edited. Set to `now()` on every author edit (D4). The client renders `(edited)` + this time (spec Q3). |
| `createdAt` | `DateTime` | `@default(now())` | Chronological position (oldest first, §6.5). |
| `updatedAt` | `DateTime` | `@updatedAt` | Touched on edit (Prisma default); display logic uses `editedAt`, never compares timestamps. |

> No `archivedAt`: frozen-ness comes from `issue.archivedAt` checked at the service (§6.4), not a per-comment flag.
> No `deletedAt`: delete physically removes the row + joins in one transaction (§6.3). Archive ≠ delete.
> No `tsvector` column in F8 — full-text over comment content lands in F10 Search (§7).

```prisma
model Comment {
  id          String    @id @default(cuid())
  workspaceId String
  issueId     String
  authorId    String
  content     String    @db.Text
  editedAt    DateTime?
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  workspace Workspace        @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  issue     Issue            @relation(fields: [issueId], references: [id], onDelete: Cascade)
  author    User             @relation("AuthoredComments", fields: [authorId], references: [id], onDelete: Restrict)
  mentions  CommentMention[]

  @@index([workspaceId])
  @@index([issueId])
  @@index([authorId])
  @@index([issueId, createdAt])
  @@map("comment")
}
```

`Workspace` gains `comments Comment[]`. `Issue` gains `comments Comment[]`. `User` gains `authoredComments Comment[] @relation("AuthoredComments")` + `commentMentions CommentMention[]`.

### 2.2 `comment_mention` — resolved mentions (the notify list)

One row per distinct mentioned user per comment. Records **only resolved mentions** — unknown tokens stay literal text in `content` and produce no rows (spec §3.3/rule 8). Dedup is structural: the composite PK makes "exactly one notification per user per comment" (spec rule 4) hold at the storage layer, not by discipline.

| Column | Type | Attr | Notes |
|---|---|---|---|
| `commentId` | `String` | FK → `comment.id` `onDelete: Cascade` | Comment delete removes its joins (issues untouched, users untouched — same unlink semantics as `issue_label`, D2). |
| `mentionedUserId` | `String` | FK → `user.id` `onDelete: Cascade` + `@@index([mentionedUserId])` | The mentioned human. `Cascade` (deliberate divergence from `authorId Restrict`, D3): if a user row is ever deleted, their mention joins vanish and the `@token` falls back to literal text; the comment itself survives. |
| `createdAt` | `DateTime` | `@default(now())` | Encounter order within the comment (first-mention order). Read path orders mentions by this. |

```prisma
model CommentMention {
  commentId       String
  mentionedUserId String
  createdAt       DateTime @default(now())

  comment       Comment @relation(fields: [commentId], references: [id], onDelete: Cascade)
  mentionedUser User    @relation("CommentMentions", fields: [mentionedUserId], references: [id], onDelete: Cascade)

  @@id([commentId, mentionedUserId])
  @@index([mentionedUserId])
  @@map("comment_mention")
}
```

No `workspaceId` denormalization — scope resolves via `comment.workspaceId` (same reasoning as `issue_label`, D2). No raw-SQL indexes needed in F8: dedup is the Prisma `@@id`, chronology is the `@@index([issueId, createdAt])` above.

`Comment` gains `mentions CommentMention[]` (above). `User` gains `commentMentions CommentMention[] @relation("CommentMentions")`.

---

## 3. Key decisions & alternatives

### D1 — Two tables: `comment` + `comment_mention` join (not parse-at-read, not array column)

**Decision:** resolved mentions are persisted as join rows at write time, mirroring `issue_label` (F5 D6).

*Considered and rejected:* (a) parse-at-read (regex `content` on every render, resolve live against members) — re-resolves history every read, so a rename/leave rewrites old conversations and duplicate-collapse must be recomputed per render; notifications would have no stable source of truth. (b) `mentionedUserIds String[]` array column — unqueryable ("all mentions of me") without Postgres array operators and outside the Prisma join conventions the codebase already uses. The join keeps module ownership clean: comments owns its facts, notifications consumes the join as the notify list.

### D2 — Join semantics = `issue_label` semantics (cascade both sides, no workspace denormalization)

**Decision:** `commentId Cascade` (comment delete unlinks, users untouched) + no `workspaceId` on the join. Same unlink contract as labels: deleting a comment never deletes users or issues.

### D3 — `authorId` is `Restrict` non-nullable; `mentionedUserId` is `Cascade` (intentional split)

**Decision:** authorship is an invariant — "every comment has one author" (spec rule 1) holds structurally because the column cannot be null and the DB blocks user-row deletion while comments reference it, mirroring `issue.creatorId` (F5 D3). Mentioned-ness is a reference, not an invariant — if a mentioned user row is ever deleted, the join vanishes and rendering falls back to literal text, mirroring the `issue_label` both-`Cascade` pattern.

*Why the split:* blocking user deletion on authored content forces correct sequencing (anonymize/transfer first — same reasoning as `creatorId Restrict`); blocking it on *being mentioned* would let any mention veto a GDPR erase. Leaving a workspace affects neither side — only the membership row is deleted; `authorId`/`mentionedUserId` point at `user.id`, so leavers' comments and historical mentions survive intact.

### D4 — Explicit `editedAt`, not derived from `updatedAt`

**Decision:** `editedAt DateTime?`, `null` until the first author edit, then `now()` per edit (locked 2026-09-04). The client renders `(edited)` + this time (spec Q3 resolved).

*Rejected:* `createdAt != updatedAt` comparison — conflates system touches with author edits and forces every reader to re-derive the flag; a nullable column states the fact once. No full edit history in MVP (spec §3.2/rule 6) — a history table is the post-MVP path if diffs are demanded.

### D5 — Content bounds 1–10,000 chars, trimmed, empty rejected (locked)

**Decision:** `commentContentSchema = z.string().trim().min(1).max(10000)` (spec Q2 resolved). Matches the `issue.description`/project-goal 10k bound so the composer, validation, and column agree. `Text` column stays unbounded — the bound lives at the Zod edge, same as descriptions.

### D6 — Handle rule: single-token `@handle`, no spaces (locked 2026-09-04)

**Decision:** token grammar `/@([A-Za-z0-9_.\-]+)/` — one word, no spaces. Resolution at write time, case-insensitive, against **current** workspace members only: a token resolves to a member when it equals (case-insensitive) the member's full `user.name` **or any whitespace-separated word of it** — so `@maya` hits `Maya Chen` and `@mchen` does not unless it is a word. Rationale: real display names contain spaces but handles cannot; word-matching keeps single-word handles usable without inventing a separate handle column in MVP.

Consequences, all deliberate: unknown tokens (including members who left — no current membership row) stay literal text with no rows and no notifications (rule 8). Duplicate tokens for the same user collapse via the `@@id` PK (rule 4). If two current members match one token (two Mayas), **both** resolve and both are notified — rare in MVP, documented; silent misses are worse than double-notifies, and a handle column is the post-MVP fix.

*Rejected:* requiring handles to equal the full multi-word name (would force spaces into handles against the locked decision); adding a `user.handle` column now (new identity surface, uniqueness + rename + migration cost for zero MVP workflow gain — revisit if ambiguity reports arrive).

### D7 — Edit silently recomputes joins, never notifies (locked)

**Decision:** every author edit re-parses `content` with the D6 rule against *current* members and replaces the join set (delete-removed, insert-added) — but emits **zero** notifications (spec §3.3/rule 4). Bob's notification from v1 stays (no take-backs); carol added in v2 gets nothing.

*Rejected:* freezing joins at create — edited text would render stale links (text says carol, link says bob). Recomputation keeps rendering truthful while the no-re-notify rule keeps it spam-free.

### D8 — Comment delete cascades its mention notifications (no dead links, locked)

**Decision:** deleting a comment removes its `comment_mention` joins **and** its mention notifications in the same transaction (§6.3) — via `notification.commentId FK Cascade` (F6 owns the column; F8 documents the contract, §7) or a same-tx `deleteForComment` service call where the FK cannot cover it. Assignment notifications are unaffected (different event, different source row).

*Rejected:* retaining mention notifications pointing at the issue (`SetNull commentId`) — the click target (comment context) is gone while the badge claims otherwise; spec's attention surface should never point at deleted content. Issue delete already cascades notifications (notifications rule 5); comment delete extends the same principle one level down.

### D9 — Archived issue freezes ALL comment writes (locked)

**Decision:** `issue.archivedAt IS NOT NULL` rejects create, edit, **and** delete on its comments (service preconditions, §6.4) — consistent with the F5 archived-issue matrix (update/attach/detach all rejected). Reads (list/detail) stay allowed; existing comments remain visible (spec §3.1).

*Considered:* rejecting only creates (the Plan's literal wording) while allowing author edit/delete — rejected: an archived issue is a frozen record; mutating its conversation (even deletions) rewrites history the archive is meant to preserve. Restore-then-edit is the way out.

### D10 — Chronological cursor over `(createdAt, id)` ASC (locked)

**Decision:** list reads order `(createdAt ASC, id ASC)` with an opaque cursor of the last row — oldest-first conversation (spec §3.1/rule 5), same cursor mechanics as `issue_history` (F5), ascending instead of the issue-list direction. Default limit 50, max 100 (api-design detail; the `@@index([issueId, createdAt])` above serves it).

---

## 4. Shared contracts (`packages/shared`)

Added in F8, consumed by `api` and `web` (ADR-002). Mirrors the Prisma models above.

```ts
// canonical bounds — match D5/D6
export const commentContentSchema = z.string().trim().min(1).max(10000);
export const mentionTokenRegex = /@([A-Za-z0-9_.\-]+)/g; // single-token handles, no spaces (D6)

// request contracts owned by the comments module
export const createCommentSchema = z.object({
  content: commentContentSchema,
  // NOTE: no mentions field — mentions are derived server-side from content (D1/D6).
  // Client may send candidate handles for suggestion UX, but the server re-parses authoritatively.
});

export const updateCommentSchema = z.object({
  content: commentContentSchema, // full replacement; editedAt set server-side (D4)
});

// response contracts
export const commentAuthorCardSchema = z.object({
  userId: z.string(),
  name: z.string(),
  email: z.string().email(),
  image: z.string().nullable(),
});

export const commentMentionCardSchema = z.object({
  userId: z.string(),
  name: z.string(),   // resolved display name at read time (falls back to literal when unresolvable)
  image: z.string().nullable(),
});

export const commentCardSchema = z.object({
  id: z.string(),
  workspaceId: z.string(),
  issueId: z.string(),
  author: commentAuthorCardSchema,
  content: z.string(),
  mentions: z.array(commentMentionCardSchema), // resolved joins in encounter order
  editedAt: z.string().datetime().nullable(),  // null ⇒ never edited (D4)
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});

// list page (chronological, cursor over (createdAt, id) ASC — D10)
export const commentListQuerySchema = z.object({
  limit: z.coerce.number().int().min(1).max(100).optional(), // default 50
  cursor: z.string().optional(),                             // opaque base64url of (createdAt, id)
});

export const commentListPageSchema = z.object({
  comments: z.array(commentCardSchema),
  nextCursor: z.string().nullable(), // null ⇒ end
});
```

List/detail query shapes and cursor encoding live in `api-design.md`; these card schemas are the single source both consume — no parallel shapes.

---

## 5. Integrity invariants → spec rule mapping

| Spec rule | Enforcement point |
|---|---|
| 1 — every comment in one issue, one author | `issueId` non-null FK `Cascade`; `authorId` non-null FK `Restrict` (D3) |
| 2 — members only; archived issues reject new comments | service: membership via `resolveWorkspaceContext` + `issue.archivedAt IS NULL` precondition; freeze-all extends to edit/delete (D9) |
| 3 — author-only edit/delete, roles never override | service authorship check `comment.authorId === session.userId` on every mutation; no role bypass in MVP |
| 4 — one notification per user per comment; edits never re-notify | `@@id([commentId, mentionedUserId])` dedup + create-tx fan-out once + zero emission on edit (D7) |
| 5 — chronological order; deleted removed | `(createdAt, id) ASC` reads (D10); physical delete, no tombstones |
| 6 — edited indicator; no full history | `editedAt` set on edit, read verbatim (D4) |
| 7 — empty rejected; long bounded | `commentContentSchema` 1–10k at the edge (D5) |
| 8 — unresolved stays literal, no error/notification | D6: parse misses produce no rows, comment still valid |

Integrity summary — constraints added in F8:

| Constraint | Where | Purpose |
|---|---|---|
| FK `comment.workspaceId → workspace` `Cascade` | `comment` | Comment dies with its workspace |
| FK `comment.issueId → issue` `Cascade` | `comment` | Comment dies with its issue (issues rule 9) |
| FK `comment.authorId → user` `Restrict` | `comment` | Authorship invariant; blocks user erase while comments exist |
| `@@id([commentId, mentionedUserId])` | `comment_mention` | Duplicate-collapse is structural (rule 4) |
| FK `comment_mention.commentId → comment` `Cascade` | `comment_mention` | Comment delete unlinks (users/issues untouched) |
| FK `comment_mention.mentionedUserId → user` `Cascade` | `comment_mention` | Mention joins vanish with a deleted user; comment survives as literal text |
| `@@index([workspaceId])` / `[issueId]` / `[authorId]` | `comment` | Workspace / conversation / by-author hot paths |
| `@@index([issueId, createdAt])` | `comment` | Chronological conversation reads + cursor pagination |
| `@@index([mentionedUserId])` | `comment_mention` | "Mentions of me" + notification-fan-out reads |

---

## 6. Lifecycle semantics at the data layer

Every multi-write operation runs in a single Prisma `$transaction` with guards re-evaluated inside the transaction. Authorship and archived checks are service preconditions — the DB carries the cascade/dedup facts, the service carries the permission facts (same split as F5).

### 6.1 Creation (atomic, spec §4.1)

```text
tx {
  assert caller is workspace member (guard chain) + issue in same workspace
  assert issue.archivedAt IS NULL else 409 ISSUE_ARCHIVED (D9)
  INSERT comment { workspaceId, issueId, authorId: caller.userId, content: trimmed }
  mentions = parse(content, D6) against current workspace members → dedup (encounter order)
  INSERT comment_mention rows for resolved users
  // F6 hook: per distinct mentioned user (excluding self-mentions — notifying yourself is a no-op):
  //   notificationsService.createMention({ workspaceId, issueId, commentId, recipientId, actorId }, tx)
} → 201 commentCard (mentions inline)
```

Self-mention (`@me` resolving to the author) creates the join row (rendering stays truthful) but emits no notification — notifying yourself is a no-op, same discipline as F5's same-person-assignee no-op. Unknown tokens create nothing and fail nothing. The comment and its notifications commit together or not at all (notifications rule 7).

### 6.2 Edit (author-only, spec §4.2)

Preconditions in-tx: `comment.authorId === caller.userId` else `403 NOT_COMMENT_AUTHOR` (roles never override — Owner/Admin included); `issue.archivedAt IS NULL` else `409 ISSUE_ARCHIVED` (D9); content passes `commentContentSchema`.

```text
tx {
  UPDATE comment SET content = trimmed, editedAt = now()
  recompute joins: DELETE removed + INSERT added per D6 vs current members (D7)
  // zero notification writes — edits never re-notify (rule 4)
} → 200 commentCard
```

Same-content edit is a no-op (no write, `editedAt` untouched — mirrors F5 no-op discipline).

### 6.3 Delete (author-only, spec §4.3)

Preconditions in-tx: authorship as above; `issue.archivedAt IS NULL` else `409 ISSUE_ARCHIVED` (D9, freeze-all).

```text
tx {
  DELETE FROM comment WHERE id = ?  // cascades comment_mention joins
  delete mention notifications for commentId  // D8 — FK Cascade or same-tx deleteForComment
} → 200 { deletedCommentId }
```

Assignment notifications, issue rows, user rows, and sibling comments are untouched. Any failure rolls back everything — the comment and its joins/notifications never half-exist.

### 6.4 Archived-issue interaction

`issue.archivedAt IS NOT NULL` freezes the conversation (D9): list/detail reads allowed (existing comments remain visible, spec §3.1); create/edit/delete all rejected with `409 ISSUE_ARCHIVED`. Restore-then-act is the way out. Orthogonal axis to the workspace freeze below.

### 6.5 Reads (chronological, spec §3.1)

Conversation reads scope to one issue: `WHERE workspaceId AND issueId` (+ `issue.archivedAt` never filters reads — archived conversations stay visible), order `(createdAt ASC, id ASC)`, cursor over the last row + `limit` (default 50, max 100). Mentions resolve per comment via the join → `user` (name/image); joins whose user is gone render as literal text from `content` (D3 fallback). No `archived=true` flag — comments have no archive dimension of their own.

### 6.6 Member leave / remove (F3 ↔ F8)

Nothing to transfer and nothing to clean: comments keep `authorId` (user row survives; only membership is deleted) and mention joins keep `mentionedUserId`. The conversation is historical fact — a leaver's messages render with their stored `user.name`, and past mentions of them stay linked. New mentions of a leaver resolve to nothing (no current membership ⇒ literal text, D6). No cross-module service call — the only F8 contract with members is the read-time membership check for write permission and mention resolution.

### 6.7 Archived-workspace interaction

`workspace.status = ARCHIVED` freezes the container (F2 guard): `GET` allowed, every comment write rejected with `409 WORKSPACE_ARCHIVED` via `resolveWorkspaceContext({ rejectArchived: true })`. Service reasserts defensively. Archived-issue freeze (§6.4) is the orthogonal per-row axis, enforced in the comments service.

---

## 7. Forward handoffs — what this model does NOT contain

| Consumer | Contract F8 provides | Landed |
|---|---|---|
| **Issues (F5, already shipped)** | `comment.issueId → issue Cascade` — closes F5 §7 (issue delete removes its comments). Archived-issue freeze honored by comments, not by issues. | **F8 implements** |
| **Notifications (F6)** | Mention-event hook: on comment create, the comments transaction exposes `[{ workspaceId, issueId, commentId, recipientId, actorId }]` (deduped, self excluded) for `notificationsService.createMention(...)` in the same tx; on comment delete, `deleteForComment(commentId, tx)` (or `notification.commentId` FK `Cascade` where F6 models it — D8). Edit emits nothing. | **F8 implements** (F6 owns the `notification` rows) |
| **Dashboard (F9)** | Reads comment queries (recent comments per workspace/issue) + `comment.createdAt` for recent-activity aggregation. No new comment columns; recently-viewed is F9's side effect. | **F9 implements** |
| **Search (F10)** | Generated `tsvector` on `comment(content)` + GIN; grouped search reads comment cards. F8 ships plain chronological reads only. | **F10 implements** |
| **Settings (F11)** | Nothing — no per-user comment preferences in MVP. | — (intentionally none) |

---

## 8. Migration workflow

Hand-modeled Prisma (like workspace/members/projects/issues/cycles):

```bash
# 1 — add Comment + CommentMention models + back-relations
#     on Workspace/Issue/User
# 2 — run
pnpm --filter @shipyard/api db:migrate -- --name add_comments_and_mentions
pnpm --filter @shipyard/api db:generate
```

- The migration produces: 2 tables (`comment`, `comment_mention`), 0 new enums, FKs + indexes above. No raw-SQL appends — Prisma expresses everything F8 needs (no functional or partial indexes; dedup is `@@id`).
- `User` relation names (`AuthoredComments`, `CommentMentions`) are explicit to avoid ambiguity with `assignedIssues`/`createdIssues`/`issueHistoryActions`.
- The F1 Testcontainers harness applies migrations automatically each test run.

**Post-migration verification (manual, once):**

```sql
-- every comment has exactly one issue and one author
SELECT id FROM comment WHERE issue_id IS NULL OR author_id IS NULL;
-- no duplicate mentions per comment (PK guarantees; confirms no legacy drift)
SELECT comment_id, mentioned_user_id, count(*) FROM comment_mention GROUP BY 1,2 HAVING count(*)>1;
-- every comment's workspace matches its issue's workspace
SELECT c.id FROM comment c JOIN issue i ON i.id = c.issue_id WHERE c.workspace_id != i.workspace_id;
-- every mention join's comment still exists (FK guarantees; sanity)
SELECT cm.comment_id FROM comment_mention cm LEFT JOIN comment c ON c.id = cm.comment_id WHERE c.id IS NULL;
```

---

## 9. What we intentionally do NOT model

| Deferred | Why |
|---|---|
| `user.handle` column / handle uniqueness | Rejected in D6 — word-matching over `user.name` covers MVP; revisit if ambiguity reports arrive. |
| Comment edit history / diffs | Spec §3.2/rule 6 out of scope — `editedAt` only (D4). |
| Tombstones / `deletedAt` | Spec §3.1: deleted comments vanish; archive covers reversible hiding at the issue level. |
| `comment.archivedAt` / per-comment archive | Lifecycle inherited from the issue (D9); no second archive axis. |
| Emoji reactions, rich text, code highlighting, attachments | Spec §6 out of scope. |
| Threads / replies (`parentCommentId`) | Spec §6 out of scope — flat chronological conversation only. |
| Comment pinning | Spec §6 out of scope. |
| Per-user mention/notification preferences | Notifications spec §6 out of scope; every resolved mention notifies. |
| `tsvector` / full-text indexes on comments | Owned by Search (F10); F8 ships plain reads. |
| Comment-level notifications for edits | Forbidden by spec rule 4 (D7). |
| notifications-table columns | Owned by F6; F8 fixes only the contract (§7). |

---

## 10. Open product questions — resolved at data layer

| Spec §7 | Decision |
|---|---|
| 1 — mention syntax | **Locked:** single-token `@handle` (`/@([A-Za-z0-9_.\-]+)/`, no spaces), resolved case-insensitively against full `user.name` or any whitespace-separated word of it among **current** members; unknown stays literal; duplicates collapse via PK; ambiguous tokens notify all matchers (D6). |
| 2 — content bounds | **Locked:** 1–10,000 chars, trimmed, empty rejected (D5). |
| 3 — edited indicator detail | **Locked:** explicit `editedAt` timestamp; client renders `(edited)` + time only (D4). No diff history. |

---

## 11. References

- Shipyard: `features/comments/spec.md`, `features/issues/data-model.md` (cascade contract §6.5/§7, `issue_label` join D6, history actor D3, cursor reads), `features/notifications/spec.md` (mention emission §3.1, atomicity rule 7, cascade rule 5), `features/members/data-model.md` (membership vs user rows, cascade convention), `features/workspace/data-model.md` (identity, cascade), `features/auth/data-model.md` (`user.name` display name), `00-architecture.md` §5/§8/§9, `ADR-001`, `ADR-002`, `Implementation Plan.md` F8
- Prisma indexes & referential actions: `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- Prisma cursor pagination: `https://www.prisma.io/docs/orm/prisma-client/queries/pagination#cursor-based-pagination`

---

*Next artifact: `api-design.md` — endpoint inventory over `comment` + `comment_mention` (create/list/detail/update/delete, chronological cursor pagination), guard chain per route (writes any member, mutations author-only), error codes (`NOT_COMMENT_AUTHOR`, `ISSUE_ARCHIVED`, mention payload shape), and the notifications-create/delete wiring.*
