# Settings — Feature Spec

**Status:** Approved
**Last updated:** 2026-08-22
**Design sources:** PRD §5.11 · UX (settings screens, view toggles, appearance) · UI 03-UI
**Technical design:** Excluded by design — produced during this feature's implementation step, driven by this behavioral spec.

---

## 1. What this feature is about

Settings owns the **user's account configuration** — and deliberately **delegates**: security flows belong to Auth, workspace details/danger zone to Workspace, members/roles/invitations to Members. Settings is a thin surface over domains that already own their rules; it never bypasses their guards.

## 2. What users can do

- View and edit their profile: display name and avatar.
- Choose their theme: light / dark / system (account-wide, applies everywhere).
- Toggle List ⇄ Kanban views for Issues and Projects, persisted per user per workspace.
- Upload an avatar image (type/size bounded).
- Navigate to delegated sections: Security (password/email change), Workspace settings (details, members, danger zone).

## 3. Main behaviors & actions

### 3.1 Profile
- Display name (bounded length) — **account-wide**: changes apply in every workspace.
- Avatar: upload a valid image; replace by uploading again; remove by clearing.
- Email is read-only here — identity changes flow through Auth only.

### 3.2 Appearance
- Theme: light / dark / system; account-wide; applies instantly; **system** follows the device preference; default = system.

### 3.3 View preferences
- One preference per (user, workspace) for Issues and one for Projects — **independent**.
- Switching List ⇄ Kanban persists the choice immediately.
- List-only subviews (Backlog / Blocked / Archived) never overwrite the stored preference (temporary lenses).

### 3.4 Delegated sections (links, not reimplementations)
| Section | Owned by |
|---|---|
| Security — password change, email change | Auth |
| Workspace — name/icon | Workspace |
| Workspace — members, roles, invitations | Members |
| Workspace — danger zone (archive/restore/delete) | Workspace |

### 3.5 Permission-aware UI
- Actions the user isn't allowed to perform are **omitted, not disabled** (no dead buttons).

## 4. User flows (high level)

1. **Profile:** settings → account → edit name / upload avatar → saved, applies everywhere.
2. **Theme:** settings → appearance → pick light/dark/system → applies instantly, persists.
3. **View toggle:** issues or projects → List ⇄ Kanban → choice saved per workspace; subviews (Backlog, Archived) don't touch it.
4. **Delegated:** settings tabs route to the owning domain's screens (security, workspace, members).

## 5. Business rules

1. Profile changes apply account-wide (no per-workspace profiles in the MVP).
2. Theme is account-wide; view preferences are per (user, workspace, kind).
3. Avatar uploads validate type/size; only signed/authorized access serves the image.
4. Email/identity changes are Auth's domain — Settings never writes email.
5. Workspace settings mutate only through Workspace/Members services — no bypass.
6. Unauthorized actions are omitted, not disabled.
7. The stored view preference applies to Issues and Projects independently; list-only subviews never overwrite it.

## 6. Out of scope (MVP)

Notification preferences, per-workspace themes, profile visibility settings, locale/language settings, custom avatar presets.

## 7. Open product questions

| # | Question | Notes |
|---|---|---|
| 1 | Avatar replacement/deletion | Replace = new upload; delete = clear image — confirm |
| 2 | Theme "system" detection | Client-side preference detection; server stores the choice only — confirm |
| 3 | Default view | List for both kinds (kanban opt-in) — confirm |
