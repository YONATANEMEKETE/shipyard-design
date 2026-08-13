# Notifications — API Design

**Module:** `apps/api/src/modules/notifications`
**Status:** Draft v0.1 — 2026-08-12
**Base URL:** `/api/v1` (proxied by Next.js; API internal-only)

---

## 1. Conventions

- **Resource:** `/api/v1/notifications` — **user-scoped, no workspace in the path** (the panel is global across workspaces, UX Decision 7).
- **Guards:** `requireSession` only; every query/action is scoped by `recipient = session user`.
- **No public create endpoint** — emission is internal-only (domain rule: no forgery path).
- **Errors:** global envelope; codes in §4. Statuses: `200` · `204` · `400` · `401` · `404` · `429`.

---

## 2. Endpoint Map

| Method | Path | Domain op | Guard |
|---|---|---|---|
| GET | `/notifications` | listNotifications | recipient |
| GET | `/notifications/unread-count` | getUnreadCount | recipient |
| PATCH | `/notifications/{id}` | markRead | recipient |
| POST | `/notifications/read-all` | markAllRead | recipient |
| DELETE | `/notifications/{id}` | deleteNotification | recipient |
| DELETE | `/notifications` | clearAll | recipient |

---

## 3. Endpoint Details

### 3.1 List — `GET /notifications`

**Query params:** `unreadOnly` (`true`) · `cursor` · `limit` (default 50, max 100).

**Responses:** `200` `{ notifications: Notification[], nextCursor }` — newest first. `Notification` = `{ id, type, issueId, issueIdentifier, workspaceId, actor: { id, name }, readAt, createdAt }`. The web renders copy from type + actor + identifier.

### 3.2 Unread count — `GET /notifications/unread-count`

**Responses:** `200` `{ unread: number }` — index-only COUNT (lifecycle §3). **This is the ~60s polling endpoint.**

### 3.3 Mark read — `PATCH /notifications/{id}`

**Body:** `{ read: true }` (only `true` accepted — 400 otherwise).

**Responses:** `200` `{ notification }` · `404` (unknown or not yours — identical).

### 3.4 Mark all read — `POST /notifications/read-all`

**Responses:** `200` `{ updated: number }` — one statement, no payload.

### 3.5 Delete one — `DELETE /notifications/{id}`

**Responses:** `204` · `404` (unknown or not yours — identical).

### 3.6 Clear all — `DELETE /notifications`

**Responses:** `204` — removes the recipient's entire history (permanent, per PRD).

---

## 4. Error Codes (notifications domain)

| Code | Status | Meaning |
|---|---|---|
| `NOTIFICATION_INVALID_INPUT` | 400 | Bad body (e.g. `read: false`) |
| `NOTIFICATION_UNAUTHORIZED` | 401 | No valid session |
| `NOTIFICATION_NOT_FOUND` | 404 | Unknown id or not the recipient (identical response) |
| `NOTIFICATION_RATE_LIMITED` | 429 | Rate limit hit |

---

## 5. Rate Limiting

| Endpoint | Limit |
|---|---|
| `GET /notifications/unread-count` | **120/min per user** (polled every ~60s — generous headroom) |
| `GET /notifications` | 120/min per user |
| `PATCH` · `read-all` · `DELETE` | 60/min per user |

---

## 6. Web Integration (consumers)

| Web surface | Endpoint(s) |
|---|---|
| Header bell + unread badge | `GET /notifications/unread-count` (TanStack Query refetch ~60s + on-focus) |
| Notifications panel (header, global) | `GET /notifications` (unread first, paginated) |
| Mark read on click | `PATCH /notifications/{id}` — navigation happens client-side to the issue |
| "Mark all as read" | `POST /notifications/read-all` |
| Delete / Clear all (confirmed) | `DELETE /notifications/{id}` · `DELETE /notifications` |
| Badge refresh after actions | Refetch unread-count on any mutation |

All calls go **through the Next proxy** (ADR-003).

---

## 7. OpenAPI & Shared Contracts

- Zod schemas in `packages/shared`: `Notification`, `NotificationListResponse`, `UnreadCountResponse`, `MarkReadInput` — shared with web + OpenAPI generation.
- `NotificationType` enum in `packages/shared` (ISSUE_ASSIGNED | ISSUE_MENTION) — the web switch-renders messages from it.

---

## 8. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Badge refresh on visibility change | Refetch on window focus + the 60s poll — confirm no extra triggers needed |
| 2 | Unread section header | Panel shows "unread" then "older" — ordering confirmed in implementation |
| 3 | Poll jitter | Add ±10s jitter to avoid thundering herd on the VPS — implementation detail |
