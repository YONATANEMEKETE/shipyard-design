# Comments — Feature Spec

**Status:** Approved
**Last updated:** 2026-08-22
**Design sources:** PRD §5.8 · §7.6 · UX Decision 6 · UX User Flow (comments)
**Technical design:** Excluded by design — produced during this feature's implementation step, driven by this behavioral spec.

---

## 1. What this feature is about

Comments owns the **discussion layer**: messages attached to issues, with authorship rights (author-only edit/delete), a chronological conversation, and `@mention` handling that feeds the Notifications feature.

## 2. What users can do

- Comment on an issue (workspace members only).
- Read the conversation, ordered chronologically.
- Mention workspace members with `@display-name`.
- Edit their own comments.
- Delete their own comments.
- See an "edited" indicator on edited comments.
- Click a mention to open the mentioned member's context (link rendering).

## 3. Main behaviors & actions

### 3.1 Writing & reading
- A comment belongs to exactly one issue; plain text with basic formatting, bounded length; empty submissions rejected.
- Comments are ordered chronologically (oldest first).
- **Archived issues cannot receive new comments** — existing comments remain visible.
- Deleted comments are removed from the conversation (no tombstones in the MVP).

### 3.2 Authorship (no role override)
- Users edit/delete **only their own** comments — Owner and Admin have no moderation power in the MVP.
- The UI hiding action buttons is a convenience; authorship is enforced server-side.
- Editing sets an "edited" indicator (no full diff history in the MVP).

### 3.3 Mentions
- Mentions written as `@display-name` inside the content.
- At write time, mentions are resolved against **current workspace members**:
  - A resolved mention → the mentioned member gets exactly one notification per comment (duplicate mentions collapse).
  - An unknown name (or a member who left) stays as **literal text** — the comment is valid; the link simply doesn't render.
- The conversation re-renders mentions as proper member links for resolved users.
- Editing a comment does **not** re-notify (notifications fire on create only).

## 4. User flows (high level)

1. **Comment:** issue detail → composer → write text (optionally @mention) → post → appears in the conversation; mentioned members notified.
2. **Edit:** own comment → edit → replaced content + "edited" indicator.
3. **Delete:** own comment → confirm → removed from the conversation.

## 5. Business rules

1. Every comment belongs to one issue and has one author.
2. Only workspace members can comment; archived issues reject new comments.
3. Users edit/delete only their own comments; roles never override authorship (in the MVP).
4. Mentioning a user generates exactly one notification per user per comment; edits do not re-notify.
5. Comments are ordered chronologically; deleted comments are removed.
6. Edited comments carry an "edited" indicator; no full history in MVP.
7. Empty submissions are rejected; long comments are bounded.
8. Unresolved mentions remain literal text — no error, no notification.

## 6. Out of scope (MVP)

Emoji reactions, rich text editor, file attachments, code blocks with syntax highlighting, comment threads/replies, comment pinning.

## 7. Open product questions

| # | Question | Notes |
|---|---|---|
| 1 | Mention syntax | `@display-name`, resolved case-insensitively — confirm |
| 2 | Content bounds | 1–10,000 chars proposed; empty rejected — confirm |
| 3 | Edited indicator detail | `(edited)` + time only — confirm |
