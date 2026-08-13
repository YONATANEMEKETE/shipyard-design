# Settings — Domain Model

**Module:** `apps/api/src/modules/settings`
**Status:** Draft v0.1 — 2026-08-12
**PRD source:** §5.11 Settings · UX decisions (view toggles, appearance)

---

## 1. Overview & Scope

Settings owns the **user's account configuration** — and deliberately **delegates** everything that already has an owner. It is the last module because it is mostly a thin service over other modules' data.

**In scope:**
- Profile: display name, avatar (via R2 upload-through-API)
- Appearance: theme (`light | dark | system`), account-wide
- View preferences: List/Kanban per user per workspace (issues + projects, independently)
- The settings **screens** (web-side navigation into delegated sections)

**Delegated (already specced — settings links, does not reimplement):**
| Section | Owner module | Endpoints |
|---|---|---|
| Security — password change / email change | auth | `POST /auth/change-password` · `POST /auth/change-email` |
| Workspace — details (name/icon) | workspace | `PATCH /workspaces/{id}` |
| Workspace — members + roles + invitations | members | `…/members/*`, `…/invitations/*` |
| Workspace — danger zone (archive/restore/delete) | workspace | `archive` · `restore` · `delete` |

---

## 2. Domain Entities

### 2.1 Profile (fields on `User`, owned by auth's table)

| Field | Notes |
|---|---|
| `name` | Display name, 1–64 chars; **account-wide** (one User row — changes apply in every workspace) |
| `image` | Avatar — R2 object URL (private bucket, signed access); optional |
| `email` | Read-only here (identity changes flow through auth) |

### 2.2 Appearance

| Field | Notes |
|---|---|
| `theme` | `light | dark | system` — **account-wide**, applies in every workspace; default `system`; stored on `User.theme` (auth data model) |

### 2.3 View Preference

Per-user, per-workspace List/Kanban choice.

| Field | Notes |
|---|---|
| `kind` | `ISSUES` · `PROJECTS` — **independent** preferences (PRD) |
| `view` | `LIST` · `KANBAN` |
| Scope | One row per (user, workspace, kind) — upserted on change |

**Invariants:**
- List-only subviews (Backlog/Blocked/Archived) never overwrite the stored preference (they are temporary lenses).
- Switching views (List ⇄ Kanban) persists the choice immediately.
- Theme changes apply instantly client-side and persist server-side.

---

## 3. Domain Invariants

1. Profile changes apply account-wide (single User row, no per-workspace profiles in MVP).
2. Theme is account-wide; view preferences are per (user, workspace, kind).
3. Avatar uploads follow the uploads-through-API pattern (validated type/size at the edge; stored in R2; only a signed URL reaches the DB).
4. Email/identity changes are auth's domain — settings never writes email.
5. Workspace settings mutate only through workspace/members services — settings never bypasses their guards.
6. Unauthorized actions in settings screens are **omitted, not disabled** (permission-aware empty states, PRD).

---

## 4. Domain Operations

| Operation | Description | Requires |
|---|---|---|
| `getProfile` / `updateProfile` | Read/update name + avatar | verified user |
| `getAppearance` / `updateAppearance` | Read/update theme | verified user |
| `getViewPreference` / `updateViewPreference` | Read/upsert per workspace+kind | workspace member |
| `uploadAvatar` | Multipart → validate → R2 → signed URL | verified user |

---

## 5. Cross-Module Contracts

| Contract | Detail |
|---|---|
| **auth** | Writes `User.name/image/theme` (table owned by auth's data model); security endpoints reused for the Security tab |
| **workspace / members** | Workspace settings tabs call their endpoints directly (web-side composition) |
| **issues / projects** | View preferences consumed at list-render time (issues module reads the preference; settings owns the storage) |
| **uploads (R2)** | Shared upload service: type/size validation (PNG/JPEG/WebP ≤ 1MB), private bucket, signed URL returned |

---

## 6. Trust Boundaries & Security Properties

1. All settings endpoints are recipient-scoped (`userId = session`); view preferences additionally require workspace membership.
2. Avatar uploads validate MIME + size at the API edge; stored server-side, served via signed URLs (no public bucket).
3. Settings never touches credentials — security flows stay inside auth's verified re-auth requirements.
4. Workspace tabs inherit workspace/members guards — no bypass path through settings.

---

## 7. Non-Goals (MVP)

Per PRD §5.11 future: custom avatars presets beyond upload, notification preferences, per-workspace themes, profile visibility settings, locale/language settings.

---

## 8. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Avatar replacement/deletion | Replacing = new upload (old R2 object orphaned — cleanup job post-MVP); deletion = clear image field — confirm |
| 2 | Theme "system" detection | Client-side `prefers-color-scheme`; server stores the choice only — confirm |
| 3 | View preference default | `LIST` for both kinds — confirm (kanban opt-in) |
