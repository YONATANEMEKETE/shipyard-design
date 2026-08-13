# Comments — Data Model

**Module:** `apps/api/src/modules/comments`
**Status:** Draft v0.1 — 2026-08-12
**Stack:** Prisma + PostgreSQL
**PRD source:** §5.8 Comments

---

## 1. Overview

Comments owns **one table** — `Comment`. Mentions are not a table: they live in the text and are parsed at write time (notifications) and read time (link rendering).

| Table | Purpose |
|---|---|
| `Comment` | The discussion message on an issue |

---

## 2. Prisma Schema

```prisma
// ============ COMMENTS MODULE ============

model Comment {
  id        String   @id @default(cuid())
  issueId   String
  authorId  String
  content   String   // 1–10,000 chars (decided); markdown-lite
  editedAt  DateTime? // set on first edit → "edited" indicator
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  issue  Issue @relation(fields: [issueId], references: [id], onDelete: Cascade)
  author User  @relation("AuthoredComments", fields: [authorId], references: [id])

  @@index([issueId, createdAt]) // chronological conversation
}
```

**No extra tables needed** — mention notifications are rows in the `notifications` module (written in the same transaction as the comment).

---

## 3. Field Notes & Design Rationale

- **`content` as plain text with markdown-lite** — no rich-text blob in the MVP (PRD defers rich text). Rendering (bold, code, links) happens client-side on the stored text; stored content stays portable and diffable.
- **`editedAt` nullable** — null = never edited; set on first edit. The UI renders `(edited)` when present. No full edit history table in the MVP (PRD: history *indicator* only).
- **No `deletedAt` tombstone** — deletion removes the row (PRD: "Deleted comments are removed from the conversation"). If moderation/audit ever needs tombstones, that's a post-MVP decision.
- **No mention columns** — resolution is derived: parse `@name` tokens → match current workspace members at write time (notification side-effect) and at read time (link rendering). This keeps the schema minimal and never drifts from the text.

---

## 4. Indexes & Constraints Summary

| Object | Type | Why |
|---|---|---|
| `Comment(issueId, createdAt)` | INDEX | Chronological conversation + cursor pagination |

The table is small and always accessed through its issue anchor — one composite index covers every query.

---

## 5. Data Lifecycle

| Event | SQL-level behavior |
|---|---|
| Create | TRANSACTION: INSERT `Comment` + INSERT mention notifications (one per distinct mentioned member, via notifications service) — both or neither |
| Edit | UPDATE `content`, `editedAt = now` (first edit) — mentions in edited text do **not** re-notify (decided: notifications fire on creation only) |
| Delete | DELETE `Comment` (author-confirmed) — removed from conversation |
| Issue deleted | CASCADE removes its comments |
| Workspace deleted | CASCADE via issue |

**Notification-on-edit decision:** editing a comment does not re-trigger mention notifications (avoids notification spam); the edit indicator tells readers something changed. Flag if you want re-notification on edit — it's a one-line change in the service.

---

## 6. Sizing & Free-Tier Fit

Comment rows ~500B–2KB. Even 100k comments ≈ 150MB worst case with indexes — inside Neon's 0.5GB free tier, and conversation pagination (50/page) keeps queries bounded.

---

## 7. Decisions Adopted (from domain model open questions)

| # | Question | Decision |
|---|---|---|
| 1 | Mention syntax | `@display-name`, case-insensitive resolution against current workspace members; unknown → literal text |
| 2 | Content bounds | **1–10,000 chars**; empty rejected |
| 3 | Edited indicator | `(edited)` + time only — no diff viewer |
