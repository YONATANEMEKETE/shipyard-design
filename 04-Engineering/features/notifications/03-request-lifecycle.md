# Notifications — Request Lifecycle

**Module:** `apps/api/src/modules/notifications`
**Status:** Draft v0.1 — 2026-08-12
**Relies on:** `02-data-model.md` · `04-api-design.md`

---

## 1. Overview

Two traffic directions with very different shapes:

1. **Emission** — never HTTP. Other modules call `notificationsService.create()` **inside their own transactions**.
2. **Consumption** — the recipient's panel + the polled unread count.

Guard chain for all consumer endpoints: `requireSession` (recipient = session user; no workspace path — the panel is global, UX Decision 7).

---

## 2. Flow — Emission (internal, not HTTP)

```
INSIDE issues transaction (assignment change):
  if assignee actually changed:
    notificationsService.create({
      userId: newAssigneeId, type: ISSUE_ASSIGNED,
      issueId, workspaceId, actorId: sessionUser
    })
  → committed with the issue update — both or neither

INSIDE comments transaction (mention resolution):
  for each DISTINCT resolved member (deduped per comment):
    notificationsService.create({ userId: memberId, type: ISSUE_MENTION,
      issueId, workspaceId, actorId: commentAuthor })
  → committed with the comment insert

NO public create endpoint exists — a notification can only exist as the
side-effect of a real event (no forgery path).
```

## 3. Flow — Polling hot path (unread count)

```
EVERY ~60s per active client (TanStack Query refetch):
  GET /api/v1/notifications/unread-count
  → requireSession
  → SELECT COUNT(*) WHERE userId = $1 AND readAt IS NULL
    (index-only scan on (userId, readAt))
  → 200 { unread: n }   [~1ms at any MVP scale]
```

**Why a dedicated endpoint:** the count is polled constantly; separating it from the full list keeps the hot path a single indexed COUNT — no pagination overhead, no payload.

## 4. Flow — Notification center

```
1. GET /api/v1/notifications?unreadOnly=&cursor=&limit=
2. requireSession → recipient-scoped query (userId = session)
3. newest first; cursor pagination (default 50)
4. 200 → { notifications: [{ id, type, issueId, issueIdentifier,
                             workspaceId, actor: { id, name }, readAt,
                             createdAt }], nextCursor }
   → web renders message from type + actor + identifier; clicking
     navigates to /workspaces/{wsId}/issues/{issueId} (archived issues
     open read-only; deleted issues can't appear — cascade)
```

## 5. Flow — Read / delete management

```
MARK ONE READ:
  PATCH /api/v1/notifications/{id} { read: true }
  → recipient check → 404 NOTIFICATION_NOT_FOUND (or not yours — identical)
  → UPDATE readAt = now                     [200 { notification }]

MARK ALL READ:
  POST /api/v1/notifications/read-all
  → UPDATE readAt = now WHERE userId = $1 AND readAt IS NULL  [200 { updated: n }]

DELETE ONE:
  DELETE /api/v1/notifications/{id}
  → recipient check → 404                   [204]

CLEAR ALL:
  DELETE /api/v1/notifications
  → DELETE WHERE userId = $1                [204]
```

## 6. Edge Cases & Failure Handling

| Case | Behavior |
|---|---|
| Reassignment to the same assignee | No event, no notification (emitter no-op) |
| User mentioned 3× in one comment | One notification (comment dedup) |
| Rapid re-assignments (A→B→A) | Notifications for B and A respectively — accepted behavior (no debounce in MVP) |
| Related issue deleted | Notification cascades away — nothing to click |
| Related issue archived | Notification remains; navigation opens read-only issue |
| Polling flood | 120/min limit + cheap indexed COUNT |
| Recipient tries another user's notification id | 404 (identical response — no existence leak) |
| Notification deleted while panel open | 404 on open → UI removes the row |
| Large unread backlog | COUNT is index-only; list is paginated — no degradation |

## 7. Dev vs Prod Differences

| Concern | Local dev | Production |
|---|---|---|
| Everything | Same behavior | Same (no external services involved) |
