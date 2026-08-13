# Comments — Domain Model

**Module:** `apps/api/src/modules/comments`
**Status:** Draft v0.1 — 2026-08-12
**PRD source:** §5.8 Comments · §7.6 Comment Rules

---

## 1. Overview & Scope

Comments owns the **discussion layer**: messages attached to issues, with authorship rights, editing/deletion rules, and mention handling.

**In scope:**
- Comment creation, editing (own), deletion (own)
- Chronological conversation display
- `@mention` parsing + the notification hand-off
- Edit history indicator

**Out of scope:**
- Notifications → `notifications` module (comments *emit* mention events, same transaction)
- Rich text, reactions, threads/replies, pinning, attachments, code blocks → post-MVP (PRD §5.8 future)
- Comments on anything other than issues (MVP: issues only — UX Decision 6)

---

## 2. Domain Entities

### 2.1 Comment

| Attribute | Notes |
|---|---|
| `content` | Plain text with basic formatting (markdown-lite); required; bounded length |
| `author` | The creating member; immutable |
| `issue` | The anchor — exactly one |
| `createdAt` / `updatedAt` / `editedAt` | `editedAt` set on first edit → "edited" indicator |
| `mentions` | **Derived** — parsed from content at write time; not stored as a relation (see §3) |

**Invariants:**
- A comment belongs to exactly one issue; a workspace member is the only possible author.
- Only workspace members may comment.
- **Archived issues cannot receive new comments** (existing ones remain visible).
- Users may edit or delete **only their own** comments — no role override: Owner and Admin have **no moderation power in the MVP** (PRD §7.6).
- Comments are ordered chronologically.
- Deleted comments are removed from the conversation (no tombstones in MVP).
- Edited comments display an "edited" indicator.

---

## 3. Mention Model

- Mentions are written as `@display-name` (or `@username` — see open question 1) inside the content.
- At write time the service **parses and resolves** mentions against **current workspace members**:
  - Resolved member → `notificationsService.notify(member, MENTION, issue)` — **same transaction** as the comment insert.
  - **Multiple mentions of the same user in one comment → one notification** (PRD edge case).
  - Unknown name / user who left the workspace → mention stays **literal text**, no notification, no error (the comment is valid; the link simply doesn't resolve).
- The create response returns the resolved `mentions: [{ userId, displayName }]` so the UI can render proper links.
- No mention table in the MVP — mentions live in the text; resolution happens at write time (notifications) and read time (link rendering). Re-parsing on read keeps rendering consistent without joins.

---

## 4. Domain Invariants

From PRD §7.6, condensed:

1. Every comment belongs to one issue and has one author.
2. Only workspace members can comment; archived issues reject new comments.
3. Users edit/delete only their own comments; workspace roles do not override authorship.
4. Mentioning a user generates exactly one notification per user per comment.
5. Comments are ordered chronologically; deleted comments are removed.
6. Edited comments carry an "edited" indicator (edit history indicator only — no full diff history in MVP).
7. Empty submissions are rejected; very long comments are bounded (open question 2).

---

## 5. Domain Operations

| Operation | Description | Requires |
|---|---|---|
| `createComment` | New comment on an active issue + mention notifications (same txn) | member |
| `listComments` | Chronological conversation (cursor pagination) | member |
| `updateComment` | Replace content (author only) → editedAt + indicator | author |
| `deleteComment` | Remove from conversation (author only, confirmed) | author |

---

## 6. Cross-Module Contracts

| Contract | Detail |
|---|---|
| **issues** | Comment belongs to `Issue` (cascade on issue delete — comments die with the issue); archived-issue gate read from issue state |
| **notifications** | Mention events written inside the comment transaction (one per user per comment) |
| **members** | Mention resolution queries current workspace members |
| **workspace** | Cascade on workspace delete (via issue) |

---

## 7. Trust Boundaries & Security Properties

1. Authorship is enforced server-side on every edit/delete — the UI hiding buttons is a convenience, not the control.
2. Mention parsing is server-side; the client can't forge a notification (it can only write text).
3. Archived-issue gate is enforced at the data layer (write rejected), not just hidden in the UI.
4. Content length and emptiness are validated at the edge (Zod).

---

## 8. Non-Goals (MVP)

Per PRD §5.8 future: emoji reactions, rich text editor, file attachments, code blocks with syntax highlighting, comment threads and replies, comment pinning.

---

## 9. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Mention syntax | `@display-name` (what users see) — recommend parsing against display names, case-insensitive; confirm |
| 2 | Content bounds | Recommend 1–10,000 chars; empty rejected |
| 3 | "Edited" indicator detail | Show `(edited)` + time only — no diff viewer in MVP; confirm |
