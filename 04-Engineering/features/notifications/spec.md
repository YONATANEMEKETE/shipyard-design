# Notifications — Feature Spec

**Status:** Approved
**Last updated:** 2026-08-22
**Design sources:** PRD §5.9 · §7.7 · UX Decision 7
**Technical design:** Excluded by design — produced during this feature's implementation step, driven by this behavioral spec.

---

## 1. What this feature is about

Notifications owns the **attention surface**: records of events that matter to a user, delivered in-app with read/unread state and an unread badge. The MVP has two event types: **assignment** (issue assigned to you) and **mention** (you were mentioned in a comment). Notifications are private per recipient.

## 2. What users can do

- See notifications in a global panel (available regardless of the workspace they're in).
- See an unread badge in the header that stays current without a manual refresh (no realtime push in MVP — the badge refreshes automatically at intervals).
- Filter unread notifications.
- Mark a notification read; mark all read.
- Delete a notification; clear all.
- Click a notification to open the related issue.
- See who triggered the event and for which issue (e.g., "Maya assigned you to SHIP-024").

## 3. Main behaviors & actions

### 3.1 Emission (what generates a notification)
- **Assignment:** when an issue is created with an assignee, or an issue's assignee **actually changes** — the new assignee gets a notification. Reassigning to the same person is a no-op (nothing fires).
- **Mention:** when a comment mentions a workspace member, each distinct mentioned member gets exactly one notification per comment (duplicate mentions of the same user collapse to one).
- No notification is generated for unassignment, status changes, invitations, ownership changes, or cycle changes in the MVP.

### 3.2 The notification center
- Private: only the recipient can see, read, or delete their notifications — no role sees another user's notifications.
- List is newest first; unread filter available.
- Read notifications remain accessible until deleted; deletion is permanent.
- Marking read only changes state — opening doesn't remove it.
- Opening a notification navigates to the related issue; archived issues remain navigable (read-only).
- If the related issue is deleted, the notification disappears with it (no dead references).

## 4. User flows (high level)

1. **Assignment:** Maya assigns you an issue → header badge increments → panel shows "Maya assigned you to SHIP-024" → click opens the issue.
2. **Mention:** comment @mentions you → one notification → click opens the issue (and the comment context).
3. **Read/clear:** mark read / mark all read / delete / clear all from the panel.
4. **Polling:** badge updates automatically while the app is open.

## 5. Business rules

1. Notifications are private and belong to exactly one recipient.
2. MVP generates notifications only for assignment/reassignment and comment mentions; assignment fires only when the assignee actually changes.
3. Read notifications remain until deleted; deletion is permanent.
4. Unread count is per recipient.
5. Related-issue deletion removes its notifications.
6. Only the recipient can act on their notifications.
7. Notification events are recorded together with their source action — an event can never exist without its source, and the source never half-exists without its event.

## 6. Out of scope (MVP)

Email/push notifications, preferences, grouping, snoozing, digests, desktop notifications, realtime push, notifications for unassignment/status changes/invitations/ownership/cycle changes.

## 7. Open product questions

| # | Question | Notes |
|---|---|---|
| 1 | Message copy | Server provides type + actor + issue; copy rendered client-side — confirm |
| 2 | Panel pagination | Unread first, then older — decide limits at implementation |
| 3 | Retention | No TTL in MVP (clear-all exists) |
