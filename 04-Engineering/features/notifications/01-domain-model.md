# Notifications — Domain Model

**Module:** `apps/api/src/modules/notifications`
**Status:** Draft v0.1 — 2026-08-12
**PRD source:** §5.9 Notifications · §7.7 Notification Rules · UX Decision 7

---

## 1. Overview & Scope

Notifications owns the **attention surface**: records of events that matter to a user, delivered in-app, with read/unread state and a polling-friendly unread count.

**In scope:**
- Notification records for the two MVP event types (assignment, mention)
- The notification center (list, read/unread filters)
- Read/unread management + delete/clear
- The **unread-count endpoint** (the ~60s polling hot path)

**Out of scope:**
- Email/push notifications, preferences, grouping, snoozing, digests → post-MVP (PRD §5.9 future)
- Event *sources* — assignment events come from `issues`, mentions from `comments`
- Realtime push — no WebSockets in MVP (ADR-001); polling covers it

---

## 2. Domain Entities

### 2.1 Notification

| Attribute | Notes |
|---|---|
| `recipient` | The user — notifications are **private**; one notification belongs to one user |
| `type` | `ISSUE_ASSIGNED` · `ISSUE_MENTION` — the only MVP types (PRD) |
| `issue` | The related item (navigation target) |
| `actor` | Who triggered the event (renders "Maya assigned you…") |
| `readAt` | Null = unread; set on read |
| `createdAt` | Event time |

**Invariants:**
- Notifications are private to the recipient; only the recipient can see/act on them.
- Only issue assignment/reassignment and comment mentions generate notifications in the MVP.
- Assignment notifications go to the **new assignee**; a reassignment to the **same** assignee is a no-op (no event, no notification).
- Read notifications remain accessible until deleted.
- Deleted notifications cannot be restored.
- Opening a notification navigates to the related issue when it still exists (archived issues remain navigable read-only).
- If the related issue is deleted, the notification dies with it (cascade — no dead references).

---

## 3. Emission Model (write contract)

Notifications are **written by other modules inside their own transactions** — the notifications module never polls or scans for events:

| Event | Emitter | When | Recipient |
|---|---|---|---|
| `ISSUE_ASSIGNED` | issues service | assignee actually changes (create-with-assignee or reassignment) | the new assignee |
| `ISSUE_MENTION` | comments service | comment created with resolved mentions | each distinct mentioned member (deduped per comment) |

- Emission = `notificationsService.create(...)` called **inside the emitter's transaction** — an event can never be recorded without its source action, and vice versa (both or neither).
- The emitter owns dedup rules (same-assignee no-op; per-comment mention dedup); the notifications module trusts its callers and stores one row per accepted event.
- No retries/queues in the MVP — synchronous writes are correct at this scale (outbox pattern is the documented post-MVP upgrade if async emission is ever needed).

---

## 4. Domain Invariants

From PRD §7.7, condensed:

1. Notifications are private to the recipient and belong to one user.
2. MVP generates notifications only for assignment/reassignment and comment mentions.
3. Read notifications remain available until deleted; deletion is permanent.
4. Marking read only changes status; opening does not remove.
5. Unread count is per recipient (the polling surface).
6. Related-issue deletion cascades the notification away.
7. Actions are recipient-only — no role can see another user's notifications.

---

## 5. Domain Operations

| Operation | Description | Requires |
|---|---|---|
| `create` (internal) | Record an event (called by issues/comments within their transactions) | internal |
| `listNotifications` | Center list (newest first, unread filter, cursor pagination) | recipient |
| `getUnreadCount` | The polling hot path — count where `readAt IS NULL` | recipient |
| `markRead` | Single notification → read | recipient |
| `markAllRead` | All unread → read | recipient |
| `deleteNotification` | Remove one (permanent) | recipient |
| `clearAll` | Remove all notifications | recipient |

---

## 6. Cross-Module Contracts

| Contract | Detail |
|---|---|
| **issues** | Emits `ISSUE_ASSIGNED` (assignment changes); `Issue.id` is the FK anchor (cascade) |
| **comments** | Emits `ISSUE_MENTION` (resolved, deduped mentions) |
| **workspace** | Workspace id stored for scoping + workspace-delete cascade |
| **web** | The notification panel is **global** (all workspaces) — UX Decision 7: "regardless of the user's current location"; each notification links into its workspace's issue |

---

## 7. Trust Boundaries & Security Properties

1. Every read/write is scoped by `recipient = session user` — enforced in queries, not just the UI.
2. The unread-count endpoint is cheap by design (one indexed COUNT) — it is polled every ~60s by every active client.
3. Notification content is structured (type + actor + issue), rendered client-side — no user-supplied HTML stored.
4. Emission is internal-only (no public "create notification" endpoint — nobody can forge a notification).

---

## 8. Non-Goals (MVP)

Per PRD §5.9 future: email notifications, push, preferences, grouping, snoozing, realtime desktop notifications, digests, and notifications for unassignment/status changes/invitations/ownership/cycle changes.

---

## 9. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Notification message copy | Rendered client-side from type + actor + issue identifier (e.g., "Maya assigned you to SHIP-024") — confirm copy lives in the web app |
| 2 | Panel pagination | 50/page cursor — unread section first, then older; confirm |
| 3 | Retention | No TTL in MVP (clear-all exists); revisit if storage grows |
