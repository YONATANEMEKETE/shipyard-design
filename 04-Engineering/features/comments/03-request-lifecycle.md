# Comments — Request Lifecycle

**Module:** `apps/api/src/modules/comments`
**Status:** Draft v0.1 — 2026-08-12
**Relies on:** workspace lifecycle §5 (guard chain) · `02-data-model.md` · `04-api-design.md`

---

## 1. Overview

Comments are a small, high-frequency surface: create (with the mention hand-off), chronological reads, and author-only edit/delete. The interesting flow is **create** — mention parsing + notification side-effects inside the comment transaction. Guard chain: `requireSession → requireWorkspaceMember`.

---

## 2. Flow — Create comment (with mentions)

```
1. POST /workspaces/{wsId}/issues/{issueId}/comments
   body: { content }                              [Zod: 1–10,000 chars, trimmed]
2. requireSession → requireWorkspaceMember
3. issue must exist and NOT be archived           [404 COMMENT_ISSUE_NOT_FOUND /
                                                   409 COMMENT_ISSUE_ARCHIVED]
4. CREATE TRANSACTION (single):
   a. INSERT Comment (author = session user)
   b. PARSE mentions from content (@display-name tokens)
   c. RESOLVE against current workspace members:
      - member found → notificationsService.notify(member, MENTION, issue)
        (DEDUPED: multiple mentions of the same user → ONE notification)
      - unknown / left the workspace → skipped silently (literal text)
   — comment + notifications commit or roll back together
5. 201 → { comment: { id, content, author, createdAt, editedAt: null,
                      mentions: [{ userId, displayName }] } }
   → UI renders the comment; mention links resolve
```

**Failure:** any step rolls back — no comment without its notifications, no notification for a comment that failed.

## 3. Flow — Conversation list

```
GET /workspaces/{wsId}/issues/{issueId}/comments?cursor=&limit=
1. guards: member
2. chronological read (oldest first — conversation order, PRD)
3. cursor pagination (default 50)
4. 200 → { comments: Comment[], nextCursor }
   — each comment carries resolved mention targets for link rendering
     (parsed at read time; no join tables)
```

## 4. Flow — Edit comment (author only)

```
1. PATCH /workspaces/{wsId}/comments/{commentId}
   body: { content }                              [Zod: same bounds]
2. guards: member
3. AUTHORSHIP CHECK: comment.authorId == session user
   → else 403 COMMENT_FORBIDDEN
   (Owner/Admin have NO override — no moderation power in MVP, PRD §7.6)
4. UPDATE content + editedAt = now (first edit only)
   — mentions in the NEW text do NOT re-notify (decision, data model §5)
5. 200 → { comment } (editedAt set → UI shows "edited")
```

## 5. Flow — Delete comment (author only)

```
1. DELETE /workspaces/{wsId}/comments/{commentId}
2. guards: member
3. AUTHORSHIP CHECK (same as edit)                [403 COMMENT_FORBIDDEN]
4. confirm dialog (web)
5. DELETE Comment — removed from the conversation (no tombstone)
6. 204
```

## 6. Edge Cases & Failure Handling

| Case | Behavior |
|---|---|
| Empty / whitespace-only content | 400 `COMMENT_INVALID_INPUT` |
| Content over 10,000 chars | 400 `COMMENT_INVALID_INPUT` |
| Comment on archived issue | 409 `COMMENT_ISSUE_ARCHIVED` (data layer) |
| Mention of a user who left the workspace | Literal text, no notification, no error |
| Mention of a non-existent name | Literal text, no notification |
| Same user mentioned 3× in one comment | ONE notification (deduped at parse time) |
| Edit someone else's comment | 403 `COMMENT_FORBIDDEN` — even for Owner/Admin |
| Delete someone else's comment | 403 `COMMENT_FORBIDDEN` |
| Owner/Admin tries to moderate a comment | 403 — no moderation override in MVP (documented PRD rule) |
| Comment's issue deleted | Comment cascades away |
| Concurrent edits | Last-write-wins (no versioning in MVP — comment edits are low-stakes) |
| Comment deleted while being edited | 404 on the edit — UI removes the row |

## 7. Dev vs Prod Differences

| Concern | Local dev | Production |
|---|---|---|
| Mention resolution | Same (members table) | Same |
| Notifications | Same in-transaction writes | Same |
| Rate limits | Same | Same |

