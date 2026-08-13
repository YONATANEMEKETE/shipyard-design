# Notifications — Data Model

**Module:** `apps/api/src/modules/notifications`
**Status:** Draft v0.1 — 2026-08-12
**Stack:** Prisma + PostgreSQL
**PRD source:** §5.9 Notifications

---

## 1. Overview

Notifications owns **one table** — `Notification` — written by other modules' transactions, read by the recipient's center + polling endpoint.

| Table | Purpose |
|---|---|
| `Notification` | One event record per recipient |

---

## 2. Prisma Schema

```prisma
// ============ NOTIFICATIONS MODULE ============

enum NotificationType {
  ISSUE_ASSIGNED
  ISSUE_MENTION
}

model Notification {
  id          String           @id @default(cuid())
  userId      String           // recipient — private to this user
  workspaceId String           // scope + workspace-delete cascade
  type        NotificationType
  issueId     String           // navigation target
  actorId     String           // who triggered the event (message rendering)
  readAt      DateTime?        // null = unread
  createdAt   DateTime         @default(now())

  user      User      @relation("ReceivedNotifications", fields: [userId], references: [id], onDelete: Cascade)
  issue     Issue     @relation(fields: [issueId], references: [id], onDelete: Cascade)
  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@index([userId, readAt])    // unread count (the polling hot path) + unread filter
  @@index([userId, createdAt]) // center list ordering + pagination
}
```

---

## 3. Field Notes & Design Rationale

- **`readAt` nullable, no `read` boolean** — null = unread; the unread COUNT is `WHERE readAt IS NULL`, served by the `(userId, readAt)` index in a single index-only scan.
- **Structured, not templated** — type + actorId + issueId stored; the message string is a *rendering* concern in the web app ("Maya assigned you to SHIP-024"). No stored HTML, no copy drift between API and UI, and future types (invitation, cycle) need no schema change.
- **`issueId` cascade** — the PRD's "navigate to the related issue when it still exists" is satisfied structurally: the notification simply doesn't exist when the issue doesn't. Archived issues keep their notifications (archive ≠ delete) and remain navigable read-only.
- **`actorId`** — enables "who did this" rendering; also useful for future grouping (post-MVP).
- **No read-tracking table, no preferences table** — both are post-MVP (PRD future list). Clear-all is a single DELETE statement.

---

## 4. Indexes & Constraints Summary

| Object | Type | Why |
|---|---|---|
| `Notification(userId, readAt)` | INDEX | Unread count (60s polling hot path) + unread filter |
| `Notification(userId, createdAt)` | INDEX | Center list ordering + cursor pagination |
| FK cascades (user, issue, workspace) | FK | Dead references never survive |

---

## 5. Data Lifecycle

| Event | SQL-level behavior |
|---|---|
| Assignment changes | Emitter transaction: INSERT `Notification` (ISSUE_ASSIGNED, new assignee) — same txn as the issue update |
| Comment mentions | Emitter transaction: INSERT `Notification` (ISSUE_MENTION) per distinct mentioned member — same txn as the comment |
| Mark read | UPDATE `readAt = now` |
| Mark all read | UPDATE `readAt = now WHERE userId = $1 AND readAt IS NULL` (one statement) |
| Delete one | DELETE by id (recipient-scoped) |
| Clear all | DELETE `WHERE userId = $1` |
| Issue deleted | CASCADE removes its notifications |
| User/workspace deleted | CASCADE |

---

## 6. Sizing & Free-Tier Fit

Notification rows ~300B. Even 500k notifications ≈ 150–200MB with indexes — inside Neon's 0.5GB free tier, and the recipient-scoped indexes keep every query (including the polled count) at O(unread count). No TTL in MVP; clear-all is the user-facing lever, and retention tuning is a documented post-MVP knob if storage grows.

---

## 7. Decisions Adopted (from domain model open questions)

| # | Question | Decision |
|---|---|---|
| 1 | Message copy | **Rendered client-side** from type + actor + issue identifier; API stores structured fields only |
| 2 | Panel pagination | 50/page cursor, newest first, unread filter supported |
| 3 | Retention | **No TTL in MVP**; clear-all available; revisit at scale |
