# Settings — Request Lifecycle

**Module:** `apps/api/src/modules/settings`
**Status:** Draft v0.1 — 2026-08-12
**Relies on:** `02-data-model.md` · `04-api-design.md` · auth/workspace/members guards (delegation)

---

## 1. Overview

Settings traffic is light and user-scoped: profile/appearance reads + writes, avatar uploads, and per-workspace view-preference upserts. Everything heavy (security, workspace management) is delegated to the owning modules — settings screens are web-side composition.

Guard chains: profile/appearance → `requireSession`; view preferences → `requireSession → requireWorkspaceMember`.

---

## 2. Flow — Update profile (name / avatar)

```
1. PATCH /api/v1/settings/profile
   body: { name?, avatar? }                    [Zod: name 1-64 trimmed; avatar = signed URL or null]
2. requireSession
3. UPDATE User (name / image) — account-wide by construction (one User row)
4. 200 → { profile: { name, email, avatar, emailVerified, theme } }
   → every workspace shell shows the new name/avatar on next render
```

## 3. Flow — Avatar upload

```
1. POST /api/v1/settings/avatar (multipart/form-data, field "file")
2. requireSession
3. edge validation: MIME ∈ {image/png, image/jpeg, image/webp} · size ≤ 1MB
   → 400 SETTINGS_INVALID_INPUT
4. PUT to R2 private bucket (random object key — no user-controlled paths)
5. generate signed URL → UPDATE User.image = url
6. 200 → { avatarUrl } — the client uses it immediately; replacement deletes
   nothing (old object orphaned; cleanup job post-MVP)
```

## 4. Flow — Theme change

```
1. PATCH /api/v1/settings/appearance { theme: light|dark|system }   [Zod]
2. requireSession → UPDATE User.theme (account-wide)
3. 200 → { theme }
   → client applies instantly (dark class on <html>); `system` resolves via
     prefers-color-scheme client-side — the server stores the choice only
```

## 5. Flow — View preference toggle

```
1. PATCH /api/v1/workspaces/{wsId}/view-preferences
   body: { kind: ISSUES|PROJECTS, view: LIST|KANBAN }              [Zod]
2. requireSession → requireWorkspaceMember
3. UPSERT ViewPreference (userId, workspaceId, kind) → view
   — one statement, row created or updated
4. 200 → { preferences: { issues: LIST|KANBAN, projects: LIST|KANBAN } }
5. Reading (issues/projects pages): the owning module fetches the preference
   at render time; missing row → default LIST (no write on read)

LIST-ONLY LENSES: Backlog/Blocked/Archived subviews never touch the stored
preference — they are temporary filters over the same list endpoint.
```

## 6. Flow — Settings screen composition (web-side)

```
/settings tabs:
  Profile      → GET/PATCH /settings/profile · POST /settings/avatar
  Appearance   → GET/PATCH /settings/appearance
  Security     → auth endpoints (change-password, change-email) — reused
  Workspace    → workspace + members endpoints (details, members, danger zone)

Guard behavior: tabs/sections the user cannot act on are OMITTED, never
disabled (permission-aware empty states) — e.g., a Member sees no Danger Zone.
```

## 7. Edge Cases & Failure Handling

| Case | Behavior |
|---|---|
| Name too long / empty | 400 `SETTINGS_INVALID_INPUT` |
| Avatar wrong type / > 1MB | 400 `SETTINGS_INVALID_INPUT` |
| R2 upload fails | 502-class error mapped to `SETTINGS_UPLOAD_FAILED`; User.image untouched |
| Invalid theme value | 400 (Zod enum) |
| View preference for a workspace you're not in | 403 (membership guard) |
| Missing preference row | Default LIST — no backfill write |
| Subview (Backlog etc.) then toggle | Stored preference unchanged until an explicit toggle |
| Member opens Danger Zone | Tab omitted (not disabled) |

## 8. Dev vs Prod Differences

| Concern | Local dev | Production |
|---|---|---|
| Avatar storage | R2 dev bucket / local stub | R2 prod bucket (private, signed URLs) |
| Everything else | Same behavior | Same |
