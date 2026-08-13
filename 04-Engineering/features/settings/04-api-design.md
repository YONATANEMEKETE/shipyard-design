# Settings — API Design

**Module:** `apps/api/src/modules/settings`
**Status:** Draft v0.1 — 2026-08-12
**Base URL:** `/api/v1` (proxied by Next.js; API internal-only)

---

## 1. Conventions

- **Resources:** user-scoped (`/settings/*`) for profile/appearance/avatar; workspace-scoped (`/workspaces/{wsId}/view-preferences`) for view preferences.
- **Guards:** `requireSession` (user-scoped) · `requireSession → requireWorkspaceMember` (view prefs).
- **Delegation:** security + workspace tabs reuse auth/workspace/members endpoints — no duplicate paths here.
- **Errors:** global envelope; codes in §4.

---

## 2. Endpoint Map

| Method | Path | Domain op | Guard |
|---|---|---|---|
| GET | `/settings/profile` | getProfile | user |
| PATCH | `/settings/profile` | updateProfile | user |
| POST | `/settings/avatar` | uploadAvatar (multipart) | user |
| GET | `/settings/appearance` | getAppearance | user |
| PATCH | `/settings/appearance` | updateAppearance | user |
| GET | `/workspaces/{wsId}/view-preferences` | getViewPreference | member |
| PATCH | `/workspaces/{wsId}/view-preferences` | updateViewPreference | member |

---

## 3. Endpoint Details

### 3.1 Profile — `GET/PATCH /settings/profile`

**GET:** `200` `{ profile: { name, email, avatar, emailVerified, theme } }`.
**PATCH body:** `{ name?: string(1..64), avatar?: string|null }` (signed URL or null).

**Responses:** `200` `{ profile }` · `400 SETTINGS_INVALID_INPUT` · `429`.

### 3.2 Avatar — `POST /settings/avatar`

**Body:** `multipart/form-data`, field `file` — PNG/JPEG/WebP, ≤ 1MB.

**Responses:** `201` `{ avatarUrl }` · `400` (type/size) · `502→SETTINGS_UPLOAD_FAILED` (R2 error).

### 3.3 Appearance — `GET/PATCH /settings/appearance`

**PATCH body:** `{ theme: "light" | "dark" | "system" }` (Zod enum).

**Responses:** `200` `{ theme }` · `400` · `429`.

### 3.4 View preferences — `GET/PATCH /workspaces/{wsId}/view-preferences`

**GET:** `200` `{ preferences: { issues: "LIST"|"KANBAN", projects: "LIST"|"KANBAN" } }` — missing rows default to LIST.
**PATCH body:** `{ kind: "ISSUES"|"PROJECTS", view: "LIST"|"KANBAN" }` — upsert.

**Responses:** `200` `{ preferences }` · `400` · `403` (not a member) · `429`.

---

## 4. Error Codes (settings domain)

| Code | Status | Meaning |
|---|---|---|
| `SETTINGS_INVALID_INPUT` | 400 | Zod validation / avatar type or size |
| `SETTINGS_UNAUTHORIZED` | 401 | No valid session |
| `SETTINGS_FORBIDDEN` | 403 | Not a workspace member (view prefs) |
| `SETTINGS_UPLOAD_FAILED` | 502 | R2 object storage error |
| `SETTINGS_RATE_LIMITED` | 429 | Rate limit hit |

---

## 5. Rate Limiting

| Endpoint | Limit |
|---|---|
| `PATCH /settings/profile` · `POST /settings/avatar` · `PATCH /settings/appearance` | 30/min per user |
| `GET /settings/*` · view-preferences | 120/min per user |
| `PATCH view-preferences` | 60/min per user |

---

## 6. Web Integration (consumers)

| Web surface | Endpoint(s) |
|---|---|
| Settings → Profile tab | `GET/PATCH /settings/profile` · `POST /settings/avatar` |
| Settings → Appearance tab | `GET/PATCH /settings/appearance` |
| Settings → Security tab | Auth endpoints (change-password, change-email — no new API) |
| Settings → Workspace tabs | Workspace + members endpoints (details, members, danger zone) |
| Issues page List/Kanban toggle | `PATCH view-preferences` + preference read at render |
| Projects page List/Kanban toggle | Same, `kind: PROJECTS` |

All calls go **through the Next proxy** (ADR-003).

---

## 7. OpenAPI & Shared Contracts

- Zod schemas in `packages/shared`: `ProfileResponse`, `UpdateProfileInput`, `AppearanceResponse`, `UpdateAppearanceInput`, `ViewPreferencesResponse`, `UpdateViewPreferenceInput` — shared with web + OpenAPI generation.
- `ViewKind` / `ViewMode` enums in `packages/shared`.

---

## 8. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Avatar cleanup | Orphaned R2 objects after replacement — cleanup job post-MVP (future worker slot) — confirm acceptable |
| 2 | Profile validation | Name regex (any printable chars, no control chars) — decide at implementation |
| 3 | Appearance GET necessity | Included for completeness + future sync; web may use cached session profile — confirm |
