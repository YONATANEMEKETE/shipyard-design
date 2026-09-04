# Notifications — Data Model

**Status:** Draft for review
**Last updated:** 2026-09-04
**Sources:** `features/notifications/spec.md` · `features/issues/data-model.md` (F5 precedent — assignment-event hook §7, cascade convention, `SetNull` actor pattern) · `features/comments/data-model.md` (F8 precedent — mention-event + retraction contract §7/D8, join-as-notify-list) · `features/members/data-model.md` (F3 precedent — partial indexes, cascade convention) · `features/workspace/data-model.md` (F2 precedent — identity, cascade) · `features/auth/data-model.md` (identity — `user` table) · `00-architecture.md` §5, §8, §9, §11 (no queues/workers — synchronous side effects, polling) · `ADR-001` (Prisma + Postgres) · `ADR-002` (shared contracts) · `Implementation Plan.md` F6
**Owner:** `apps/api` — Prisma-owned (hand-modeled, like workspace/members/projects/issues/cycles/comments).

> **Locked calls (2026-09-04, spec-backed defaults):** reads are global per recipient (not `:slug`-scoped); emission is internal-only (no `POST /notifications`); self-notifications suppressed for both types; cards live-join issue/actor (no snapshots); `readAt`-only rows (no `updatedAt`); copy rendered client-side; panel cursor `(createdAt, id)` DESC + `unreadOnly` filter + dedicated unread-count; `commentId` nullable `Cascade` owned here (closes F8 D8).

---

## 1. Overview

Notifications owns the **attention surface**: private per-recipient records of the two MVP events — **assignment** (issue assigned to you) and **mention** (you were mentioned in a comment) — with read/unread state and a pollable unread count. Emission is never user-invoked: rows are written only as synchronous side effects inside the source transaction (issues create/reassign, comment create) and removed only by the recipient or by source cascade.

One new table, one new enum:

| Table / Change | Purpose | Formalized by |
|---|---|---|
| `notification` | The event record: scope, recipient, actor, source refs, type, read state | **F6 (this milestone)** |
| `NotificationType` | `ASSIGNMENT \| MENTION` (additive for future types) | **F6** |

F5 and F8 defined their sides of the contracts already — F6 implements the receiving side: `createAssignment`/`createMention` in-tx writers plus `deleteForComment` retraction (§6.1/§6.2/§7). No other module writes this table, ever.

---

## 2. Core schema (Prisma-owned)

### 2.1 `notification`

One row per recipient per event. A reassignment `A→B→A` correctly yields two rows (each was an actual change); a duplicate `@maya @maya` yields one (deduped upstream by the `comment_mention` PK before the fan-out reaches this table).

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK `@default(cuid())` | Immutable internal identifier. URL never addresses it alone except per-row read/delete routes. |
| `workspaceId` | `String` | FK → `workspace.id` `onDelete: Cascade` + `@@index([workspaceId])` | Scope for cascade + navigation + optional per-workspace filtering. Workspace delete removes its notifications (workspace spec rule 4). |
| `recipientId` | `String` | FK → `user.id` `onDelete: Cascade` + `@@index([recipientId])` | The human this is for (rule 1 — private, exactly one recipient). User delete removes their notifications; accounts outlive everything else. |
| `actorId` | `String?` | FK → `user.id` `onDelete: SetNull` + `@@index([actorId])` | Who triggered it. `SetNull` preserves the recipient's record if the actor is later deleted (same reasoning as `issue_history.actorId`, D5); card renders a "former member" fallback. |
| `issueId` | `String` | FK → `issue.id` `onDelete: Cascade` + `@@index([issueId])` | The related issue (every MVP event is issue-anchored). Issue delete removes its notifications — no dead references (rule 5). |
| `commentId` | `String?` | FK → `comment.id` `onDelete: Cascade` + `@@index([commentId])` | `null` for assignments, set for mentions. Comment delete retracts its mention notifications in the same tx — no dead links (closes F8 D8, D4). |
| `type` | `NotificationType` | — | `ASSIGNMENT \| MENTION`. The only branch the client copies on (spec Q1). |
| `readAt` | `DateTime?` | — | `null` ⇒ unread. Mark-read sets `now()`; no unread action in MVP (§6.3). Partial unread index serves the badge poll (D7). |
| `createdAt` | `DateTime` | `@default(now())` | Event order — panel is newest-first (§6.6). |

> No `updatedAt` — rows are immutable except the `readAt` flag (D6). Read-state change is not an edit worth versioning.
> No snapshot columns (`issueTitle`, `identifier`, actor name) — cards join live at read (D3). Archived issues stay navigable; renames flow through; deleted issues cascade away instead of going stale.
> No `tsvector` — notifications are never searched (F10 reads issues/projects/cycles/members only, §7).

```prisma
enum NotificationType {
  ASSIGNMENT
  MENTION
}

model Notification {
  id          String           @id @default(cuid())
  workspaceId String
  recipientId String
  actorId     String?
  issueId     String
  commentId   String?
  type        NotificationType
  readAt      DateTime?
  createdAt   DateTime         @default(now())

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  recipient User      @relation("ReceivedNotifications", fields: [recipientId], references: [id], onDelete: Cascade)
  actor     User?     @relation("SentNotifications", fields: [actorId], references: [id], onDelete: SetNull)
  issue     Issue     @relation(fields: [issueId], references: [id], onDelete: Cascade)
  comment   Comment?  @relation(fields: [commentId], references: [id], onDelete: Cascade)

  @@index([workspaceId])
  @@index([recipientId])
  @@index([recipientId, createdAt])
  @@index([issueId])
  @@index([commentId])
  @@map("notification")
}
```

Polling hot path (raw SQL, same precedent as `workspace_single_owner` / `invitation_single_pending`):

```sql
CREATE INDEX notification_unread_idx
  ON notification (recipient_id, created_at DESC) WHERE read_at IS NULL;
```

`Workspace` gains `notifications Notification[]`. `Issue` gains `notifications Notification[]`. `Comment` gains `notifications Notification[]`. `User` gains `receivedNotifications Notification[] @relation("ReceivedNotifications")` + `sentNotifications Notification[] @relation("SentNotifications")`.

Cascade convergence note (same as `issue_label` F5 §2.3): deleting an issue reaches notifications via two paths — direct (`issueId Cascade`) and indirect (`issue → comment Cascade → notification commentId Cascade`). Both converge on the same rows; either suffices, neither double-deletes. Assignment rows (no `commentId`) clean via the direct path; mention rows via both.

---

## 3. Key decisions & alternatives

### D1 — One table + type enum (not per-type tables)

**Decision:** assignments and mentions share one table branched by `type`. Both lifecycles are identical (emit in-tx → poll newest-first → mark read → delete/clear → cascade with source). Per-type tables would duplicate every read/delete route and the polling indexes for zero MVP divergence.

*Future types* (status changes, invites — all explicitly out of MVP per spec §6) widen the enum additively with a nullable source ref each, same pattern as `IssueHistoryEvent += CYCLE_CHANGED`.

### D2 — Reads are global per recipient (not `:slug`-scoped)

**Decision:** list/count/detail routes live at `/api/v1/notifications...` scoped by `recipientId = session.userId`, never by URL workspace (spec §2: "available regardless of the workspace they're in"). `workspaceId` stays on the row for cascade, navigation (`/w/:slug/issues/:id`), and optional `?workspaceId=` filtering — but it is never proof of access here; recipient ownership is.

*This is the one deliberate divergence from every prior module's guard chain,* and it is spec-mandated: the bell follows the human, not the workspace. Cross-workspace leak testing inverts accordingly — a foreign `notificationId` returns `404 NOTIFICATION_NOT_FOUND` regardless of workspace (api-design).

### D3 — Live-join cards, no denormalized snapshots (locked)

**Decision:** the card carries `actor{name,image}` + `issue{id,identifier,title,workspaceSlug,archivedAt}` resolved at read time. No `issueTitle`/`actorName` snapshot columns.

*Rejected:* snapshotting at emit ("Maya assigned you to *Old Title*") — freezes stale titles into the panel, doubles every rename write, and contradicts spec Q1 ("server provides type + actor + issue; copy rendered client-side"). The notification is a pointer to a living issue, not a frozen receipt. Deleted issues vanish via cascade instead of lingering as snapshots (rule 5).

### D4 — `commentId` nullable `Cascade` owned here (closes F8 D8)

**Decision:** F6 owns the column F8 coded against: mentions set it, assignments leave it `null`, comment delete cascades through it. The comments service additionally calls `deleteForComment` in-tx as the explicit path where the FK alone cannot express intent (bulk issue-delete already cascades both legs — §6.5).

### D5 — `recipientId Cascade` / `actorId SetNull` split (mirrors F5 D3)

**Decision:** the recipient owns the row's existence (user delete ⇒ rows gone — a deleted account has no panel), the actor merely annotates it (user delete ⇒ `actorId` nulled, row survives with a "former member" fallback). Same split as `issue.assigneeId SetNull` vs `issue.creatorId Restrict`, applied to the attention domain: erase-me must clear my inbox but must not rewrite others'.

### D6 — `readAt`-only mutability, no `updatedAt`, no mark-unread

**Decision:** the only legal mutation is unread→read (`readAt = now()`); re-marking read is idempotent (keeps the first `readAt`); deletion is the only other exit (rule 3). No `updatedAt` column — nothing else about the row ever changes, so versioning it is noise. No mark-unread endpoint in MVP (spec §2 lists read/all-read/delete/clear-all only).

### D7 — Polling indexes incl. partial unread index

**Decision:** `@@index([recipientId, createdAt])` serves the newest-first panel walk; the partial `notification_unread_idx ... WHERE read_at IS NULL` serves the ~60s badge poll without scanning read history (arch §11 — polling, no WebSockets). Raw-SQL partial index, same migration pattern as F2/F3.

### D8 — Self-notifications suppressed for both types (locked)

**Decision:** assigning an issue to yourself and mentioning yourself emit nothing — same discipline as F8 self-mention and F5 same-person-assignee no-ops. Rationale: the dominant self-assign flow (create + take it yourself) would badge-spam every author; the panel is for *other humans'* attention demands. The source rows still write normally; only the fan-out skips `recipientId === actorId`.

### D9 — Emission internal-only, same-tx (rule 7)

**Decision:** no `POST /notifications` exists. The only writers are `createAssignment` (called by issues create/reassign txs) and `createMention` (called by the comment-create tx), each executing inside the caller's `$transaction`: "an event can never exist without its source, and the source never half-exists without its event." Emission preconditions (actual-change, same-workspace, non-archived where applicable) are evaluated by the *source* service; F6 validates recipient-liveness defensively in-tx.

---

## 4. Shared contracts (`packages/shared`)

Added in F6, consumed by `api` and `web` (ADR-002). Event (internal) and card (HTTP) shapes are distinct on purpose — events are service-call arguments, cards are poll responses.

```ts
// zod enum — mirrors Prisma enum §2
export const notificationTypeSchema = z.enum(["ASSIGNMENT", "MENTION"]);

// internal event contracts (service-to-service, never HTTP bodies — D9)
export const assignmentEventSchema = z.object({
  workspaceId: z.string(),
  issueId: z.string(),
  newAssigneeId: z.string(),  // actual-change + non-self already checked by caller (D8)
  actorId: z.string(),
});

export const mentionEventSchema = z.object({
  workspaceId: z.string(),
  issueId: z.string(),
  commentId: z.string(),
  recipientId: z.string(),    // deduped upstream via comment_mention PK; self already excluded (D8)
  actorId: z.string(),
});

// response contracts (HTTP reads — live joins, D3)
export const notificationActorCardSchema = z.object({
  userId: z.string(),
  name: z.string(),   // "Former member" fallback when actorId IS NULL (D5)
  image: z.string().nullable(),
});

export const notificationIssueCardSchema = z.object({
  id: z.string(),
  identifier: z.string(), // SHIP-###, rendered verbatim
  title: z.string(),
  workspaceId: z.string(),
  workspaceSlug: z.string(),          // navigation target /w/:slug/issues/:id
  archivedAt: z.string().datetime().nullable(), // archived ⇒ read-only landing (spec §3.2)
});

export const notificationCardSchema = z.object({
  id: z.string(),
  workspaceId: z.string(),
  type: notificationTypeSchema,
  actor: notificationActorCardSchema.nullable(),
  issue: notificationIssueCardSchema,
  commentId: z.string().nullable(),   // set for MENTION (comment-context scroll); null for ASSIGNMENT
  readAt: z.string().datetime().nullable(), // null ⇒ unread (D6)
  createdAt: z.string().datetime(),
});

// panel page (newest-first cursor over (createdAt, id) DESC)
export const notificationListQuerySchema = z.object({
  unreadOnly: z.coerce.boolean().optional(),              // default false
  workspaceId: z.string().cuid().optional(),              // optional per-workspace filter (D2)
  limit: z.coerce.number().int().min(1).max(100).optional(), // default 25
  cursor: z.string().optional(),                           // opaque base64url of (createdAt, id)
});

export const notificationListPageSchema = z.object({
  notifications: z.array(notificationCardSchema),
  nextCursor: z.string().nullable(), // null ⇒ end
});

export const unreadCountSchema = z.object({
  unreadCount: z.number().int().nonnegative(),
});
```

Copy (`"Maya assigned you to SHIP-024"`) is rendered client-side from `type + actor + issue` (spec Q1) — the API never ships pre-baked sentences.

---

## 5. Integrity invariants → spec rule mapping

| Spec rule | Enforcement point |
|---|---|
| 1 — private, exactly one recipient | `recipientId` non-null FK; every read/write scopes `recipientId = session.userId` (D2) |
| 2 — only assignment-actual-change + mention-create emit | source-service preconditions in-tx (issues same-person no-op; comments edit-emits-nothing); F6 is a dumb fan-out inside the caller tx (D9) |
| 3 — reads persist until deleted; deletion permanent | `readAt` flag only (D6); physical row delete, no trash |
| 4 — unread count per recipient | `COUNT WHERE recipientId AND readAt IS NULL` via partial index (D7) |
| 5 — issue delete removes its notifications | `issueId Cascade` (+ indirect comment path converging, §2) |
| 6 — only the recipient acts | recipient-scope on every route (api-design); no role bypass — Owner cannot read another's inbox |
| 7 — event + source atomic | writers execute inside the source `$transaction` only (D9); no standalone create path exists to violate it |

Integrity summary — constraints added in F6:

| Constraint | Where | Purpose |
|---|---|---|
| FK `notification.workspaceId → workspace` `Cascade` | `notification` | Dies with its workspace |
| FK `notification.recipientId → user` `Cascade` | `notification` | Inbox dies with the account (D5) |
| FK `notification.actorId → user` `SetNull` | `notification` | Survives actor deletion with fallback (D5) |
| FK `notification.issueId → issue` `Cascade` | `notification` | No dead refs (rule 5) |
| FK `notification.commentId → comment` `Cascade` | `notification` | Comment delete retracts mentions (D4) |
| `@@index([recipientId])` / `[recipientId, createdAt]` | `notification` | Panel walk hot path |
| `notification_unread_idx` partial | `notification (recipient_id, created_at) WHERE read_at IS NULL` | Badge-poll hot path (D7) |
| `@@index([issueId])` / `[commentId]` / `[workspaceId]` | `notification` | Cascade + filter hot paths |

---

## 6. Lifecycle semantics at the data layer

All writes run inside the source `$transaction`. F6 exposes three internal functions and owns four recipient actions (routes in `api-design.md`); it owns no lifecycle of its own beyond read/unread/delete.

### 6.1 Assignment emission (F5 ↔ F6)

Called by the issues service inside the issue-create / issue-update transaction, **only** when the assignee actually changes (`old !== new`, `new != null`) and `new !== actor` (D8 self-suppress):

```text
tx {
  ... issue INSERT/UPDATE + history rows (F5) ...
  notificationsService.createAssignment({ workspaceId, issueId, newAssigneeId, actorId }, tx)
    // INSERT notification { workspaceId, recipientId: newAssigneeId, actorId, issueId,
    //                       commentId: NULL, type: ASSIGNMENT, readAt: NULL }
}
```

Unassignment (`new = null`) emits nothing; same-person set is filtered before the call (F5 rule 13 — no-op, no write, no history, no event). Create-with-assignee emits one (it is an actual change from unset). Reassignment `A→B` notifies B only — A receives nothing (no "you were unassigned" in MVP, spec §3.1).

### 6.2 Mention emission + retraction (F8 ↔ F6)

Create path — called by the comments service inside the comment-create transaction, once per distinct resolved recipient (deduped upstream by the `comment_mention` PK), self excluded (D8):

```text
tx {
  ... comment INSERT + comment_mention rows (F8) ...
  for each recipientId ≠ actorId:
    notificationsService.createMention({ workspaceId, issueId, commentId, recipientId, actorId }, tx)
}
```

Edit path — nothing. The comments edit tx calls no F6 function (rule: edits never re-notify). Delete path — retraction in the same tx as the comment delete (D4/D8):

```text
tx {
  DELETE comment → cascades comment_mention joins (F8)
  notificationsService.deleteForComment(commentId, tx)
    // DELETE notification WHERE commentId = ? (FK Cascade covers it where modeled;
    //  explicit call is the intent-readable path + covers bulk issue-delete ordering)
}
```

### 6.3 Read flows (recipient-only)

- **Mark read:** `UPDATE notification SET readAt = now() WHERE id AND recipientId AND readAt IS NULL` — first-read timestamp wins; re-mark is a no-op `200` (idempotent, D6).
- **Mark all read:** `UPDATE ... SET readAt = now() WHERE recipientId AND readAt IS NULL` (optionally `AND workspaceId = ?`) — returns `{ markedCount }`. Single statement, no row iteration.
- Both are allowed regardless of workspace archived state (reads are never frozen) and regardless of issue archived state (archived issues remain navigable, spec §3.2).

### 6.4 Delete flows (recipient-only, permanent)

- **Delete one:** `DELETE WHERE id AND recipientId` → `200 { deletedNotificationId }`. Foreign ids (another recipient's row, random cuid) ⇒ `404 NOTIFICATION_NOT_FOUND` — indistinguishable by design (D2 inverted leak test).
- **Clear all:** `DELETE WHERE recipientId` (optionally `AND workspaceId = ?` / `AND readAt IS NOT NULL`?) — base contract clears everything for the recipient; filtered variants are api-design query detail. Returns `{ deletedCount }`. No TTL exists (spec Q3) — clear-all is the retention story.

### 6.5 Cascade flows (no recipient action — source deletions)

| Source delete | Effect | Path |
|---|---|---|
| Issue delete (F5 #7 tx) | its assignment + mention rows gone | `issueId Cascade` (+ comment leg converging, §2) |
| Comment delete (F8 #5 tx) | its mention rows gone; sibling comments' rows + all assignment rows untouched | `commentId Cascade` / `deleteForComment` (D4) |
| Workspace delete | all its rows gone (every recipient's) | `workspaceId Cascade` |
| Recipient user delete | their inbox gone | `recipientId Cascade` (D5) |
| Actor user delete | rows survive, `actorId` nulled | `actorId SetNull` + "former member" fallback (D5) |

### 6.6 Reads (panel + badge + detail)

- **Panel:** `WHERE recipientId (= session) [AND readAt IS NULL] [AND workspaceId = ?]`, order `(createdAt DESC, id DESC)`, cursor over the last row, default limit 25 / max 100. Joins issue (live card incl. `archivedAt`), actor (nullable fallback), workspace slug for navigation. Read and unread rows intermix by recency unless `unreadOnly` is set (spec §3.2: newest first + unread filter — no "unread pinned first" second ordering in MVP).
- **Badge:** `COUNT(*) WHERE recipientId AND readAt IS NULL` via the partial index (D7) — the ~60s poll target, deliberately cheap and separate from the panel query.
- **Detail:** single-row read for permalink/deep-link validation; still recipient-scoped (`404` for foreign rows). Navigation target is the issue (plus `#comment-<id>` scroll for mentions); archived issues land read-only.

### 6.7 Archived-workspace / archived-issue interaction

- Archived workspace: reads (panel/badge/detail) allowed globally as usual; **no new rows can arise** inside it because every source write is frozen by `rejectArchived` guards upstream. F6 needs no archived check of its own — defense in depth asserts `workspace.status` on the internal writers and skips emission (never errors; the source tx would already have failed).
- Archived issue: rows survive and stay navigable read-only (spec §3.2); assignment changes and new comments are frozen upstream, so no new rows reference archived issues except reopen-then-act flows.

---

## 7. Forward handoffs — what this model does NOT contain

| Consumer | Contract F6 provides | Landed |
|---|---|---|
| **Issues (F5, already shipped)** | `createAssignment(event, tx)` — closes F5 §7. Same-person/unassign/self filtered by the caller; F6 trusts-but-verifies liveness in-tx. | **F6 implements** |
| **Comments (F8, already shipped)** | `createMention(event, tx)` + `deleteForComment(commentId, tx)` (+ `commentId` FK) — closes F8 §7/D8. | **F6 implements** |
| **Cycles (F7)** | Nothing — explicitly no cycle-change notifications (spec §3.1). | — (intentionally none) |
| **Dashboard (F9)** | Nothing new — the header badge/bell is this module's own poll, not a dashboard aggregate. Dashboard reads issues/cycles/projects; it never reads this table. | — (intentionally none) |
| **Search (F10)** | Nothing — notifications are never searched. | — (intentionally none) |
| **Settings (F11)** | Nothing — no preferences in MVP (spec §6). A `notification_preference` table is the post-MVP path if opt-outs arrive. | — (intentionally none) |

---

## 8. Migration workflow

Hand-modeled Prisma (like workspace/members/projects/issues/cycles/comments):

```bash
# 1 — add NotificationType enum + Notification model + back-relations
#     on Workspace/Issue/Comment/User
# 2 — run
pnpm --filter @shipyard/api db:migrate -- --name add_notifications
pnpm --filter @shipyard/api db:generate
```

- The migration produces: 1 table (`notification`), 1 enum (`NotificationType`), FKs + indexes above.
- One raw-SQL append in the same migration folder (Prisma cannot express it — same pattern as `workspace_single_owner`, `invitation_single_pending`, `cycle_no_overlap`):
  1. `CREATE INDEX notification_unread_idx … WHERE read_at IS NULL` (D7)
- Emission/retraction call sites land in the issues/comments services in the same milestone (F5/F8 code changes calling the new F6 functions — no separate migration).
- The F1 Testcontainers harness applies migrations automatically each test run.

**Post-migration verification (manual, once):**

```sql
-- one recipient per row, always
SELECT id FROM notification WHERE recipient_id IS NULL;
-- every row points at a live issue (FK guarantees; sanity)
SELECT n.id FROM notification n LEFT JOIN issue i ON i.id = n.issue_id WHERE i.id IS NULL;
-- mention rows always carry a comment; assignments never do
SELECT id, type, comment_id FROM notification
  WHERE (type = 'MENTION' AND comment_id IS NULL) OR (type = 'ASSIGNMENT' AND comment_id IS NOT NULL);
-- unread partial index exists and matches the badge query shape
SELECT indexname FROM pg_indexes WHERE indexname = 'notification_unread_idx';
```

---

## 9. What we intentionally do NOT model

| Deferred | Why |
|---|---|
| `POST /notifications` / external producers | Forbidden by D9 — rule-7 atomicity requires source-tx emission; a public create would mint sourceless events. |
| Snapshot columns (`issueTitle`, `actorName`) | Rejected in D3 — live joins; renames flow, deletes cascade. |
| `updatedAt` / mark-unread | Rejected in D6 — flag-only rows; spec lists no unread action. |
| Per-type tables | Rejected in D1 — shared lifecycle, enum branches. |
| Email / push / desktop delivery | Spec §6 out of scope — in-app + ~60s poll only (arch §11). |
| Preferences / opt-outs / grouping / snoozing / digests | Spec §6 out of scope — every resolved event notifies (minus D8 self-suppress). |
| Notifications for unassignment, status, invites, ownership, cycles | Spec §3.1 closed list — no hooks, no rows. |
| TTL / retention job | Spec Q3 resolved: none — clear-all is the retention story. |
| Search over notifications | F10 excludes this table (§7). |
| Real-time push / WebSockets / queues | Arch §11 — polling is the MVP delivery story. |

---

## 10. Open product questions — resolved at data layer

| Spec §7 | Decision |
|---|---|
| 1 — message copy | **Locked:** server ships `type + actor + issue (+ commentId)`; all copy rendered client-side from the card (D3, §4). No pre-baked sentences in rows. |
| 2 — panel pagination | **Locked:** cursor over `(createdAt, id)` DESC, newest-first, `unreadOnly` filter, default 25 / max 100, plus a dedicated lightweight unread-count query for the poll loop (§6.6, D7). |
| 3 — retention | **Locked:** no TTL, no cleanup job — read rows persist until deleted/cleared (§6.4). |

---

## 11. References

- Shipyard: `features/notifications/spec.md`, `features/issues/data-model.md` (assignment hook §7, cascade convention, actor `SetNull` D3), `features/comments/data-model.md` (mention contract §7, retraction D8, join-as-notify-list D1), `features/members/data-model.md` (partial-index pattern, cascade convention), `features/workspace/data-model.md` (identity, cascade), `features/auth/data-model.md` (identity — `user` table), `00-architecture.md` §5/§8/§9/§11, `ADR-001`, `ADR-002`, `Implementation Plan.md` F6
- Prisma indexes & referential actions: `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- Prisma cursor pagination: `https://www.prisma.io/docs/orm/prisma-client/queries/pagination#cursor-based-pagination`
- PostgreSQL partial indexes: `https://www.postgresql.org/docs/current/indexes-partial.html`

---

*Next artifact: `api-design.md` — global endpoint inventory (`GET` panel, `GET` unread-count, per-row read/delete, mark-all-read, clear-all; no create route), recipient-scope guard chain (the D2 divergence), error codes, polling sequences, and the F5/F8 in-tx call-site wiring.*
