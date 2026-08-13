# Comments — API Design

**Module:** `apps/api/src/modules/comments`
**Status:** Draft v0.1 — 2026-08-12
**Base URL:** `/api/v1` (proxied by Next.js; API internal-only)

---

## 1. Conventions

- **Resources:** conversation is issue-scoped (`/issues/{issueId}/comments`); edits/deletes are comment-scoped (`/comments/{commentId}`).
- **Guards:** `requireSession → requireWorkspaceMember`; authorship checks are per-operation (never role-based).
- **Mentions:** detected server-side; returned in responses for link rendering.
- **Errors:** global envelope; codes in §4. Statuses: `200` · `201` · `204` · `400` · `401` · `403` · `404` · `409` · `429`.

---

## 2. Endpoint Map

| Method | Path | Domain op | Guard |
|---|---|---|---|
| GET | `/workspaces/{wsId}/issues/{issueId}/comments` | listComments | member |
| POST | `/workspaces/{wsId}/issues/{issueId}/comments` | createComment | member |
| PATCH | `/workspaces/{wsId}/comments/{commentId}` | updateComment | **author** |
| DELETE | `/workspaces/{wsId}/comments/{commentId}` | deleteComment | **author** |

---

## 3. Endpoint Details

### 3.1 Create — `POST /workspaces/{wsId}/issues/{issueId}/comments`

**Body (Zod):** `{ content: string(1..10000) }`

**Responses:**
- `201` — `{ comment: { id, content, author: { id, name }, createdAt, editedAt, mentions: [{ userId, displayName }] } }`
- `400` — `COMMENT_INVALID_INPUT` (empty / too long)
- `404` — `COMMENT_ISSUE_NOT_FOUND` (issue unknown or not in workspace)
- `409` — `COMMENT_ISSUE_ARCHIVED` (archived issues reject new comments)
- `429` — `COMMENT_RATE_LIMITED`

**Behavior:** mention parsing + one notification per distinct mentioned member, same transaction (lifecycle §2).

### 3.2 List — `GET /workspaces/{wsId}/issues/{issueId}/comments`

**Query params:** `cursor` · `limit` (default 50, max 100).

**Responses:** `200` `{ comments: Comment[], nextCursor }` — **chronological (oldest first)**, each with resolved `mentions`.

### 3.3 Update — `PATCH /workspaces/{wsId}/comments/{commentId}`

**Body:** `{ content: string(1..10000) }`

**Responses:**
- `200` — `{ comment }` (editedAt set on first edit → UI shows "edited"; no re-notification)
- `400` — `COMMENT_INVALID_INPUT` · `403` — `COMMENT_FORBIDDEN` (not the author) · `404` — `COMMENT_NOT_FOUND`

### 3.4 Delete — `DELETE /workspaces/{wsId}/comments/{commentId}`

**Responses:** `204` · `403` (not the author — including Owner/Admin) · `404`.

---

## 4. Error Codes (comments domain)

| Code | Status | Meaning |
|---|---|---|
| `COMMENT_INVALID_INPUT` | 400 | Empty / over-length content |
| `COMMENT_UNAUTHORIZED` | 401 | No valid session |
| `COMMENT_FORBIDDEN` | 403 | Not the author (no role override) |
| `COMMENT_NOT_FOUND` | 404 | Unknown comment id |
| `COMMENT_ISSUE_NOT_FOUND` | 404 | Issue unknown or not in workspace |
| `COMMENT_ISSUE_ARCHIVED` | 409 | New comment on an archived issue |
| `COMMENT_RATE_LIMITED` | 429 | Rate limit hit |

---

## 5. Rate Limiting

| Endpoint | Limit |
|---|---|
| `POST …/comments` | 60/min per user |
| `PATCH/DELETE …/comments/{id}` | 30/min per user |
| `GET …/comments` | 120/min per user |

---

## 6. Web Integration (consumers)

| Web surface | Endpoint(s) |
|---|---|
| Comments section (Issue Details) | `GET …/issues/{id}/comments` (chronological, paginated) |
| Comment composer (+ `@` mention suggestions — member list from members directory) | `POST …/issues/{id}/comments` |
| Inline edit | `PATCH …/comments/{id}` |
| Delete (confirmed) | `DELETE …/comments/{id}` |
| Mention links | `mentions` array from any comment response |

All calls go **through the Next proxy** (ADR-003).

---

## 7. OpenAPI & Shared Contracts

- Zod schemas in `packages/shared`: `CreateCommentInput`, `UpdateCommentInput`, `Comment`, `CommentListResponse` — shared with web + OpenAPI generation.
- The mention type (`MentionTarget = { userId, displayName }`) is shared with the notifications module's contract.

---

## 8. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Mention suggestion UX | Composer queries the member directory (existing endpoint) for `@` autocomplete — confirm debounce/filter details in implementation |
| 2 | Linkify URLs in content | Client-side rendering concern — no server schema impact; confirm during UI implementation |
| 3 | Comment count on issue cards | Cheap `COUNT` join on the list query — decide in implementation whether card badges show counts (PRD doesn't require it) |
