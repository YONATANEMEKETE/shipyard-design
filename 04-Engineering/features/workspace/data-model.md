# Workspace — Data Model

**Status:** Approved for implementation (F2) — open questions resolved with product owner 2026-08-27
**Last updated:** 2026-08-27
**Sources:** `features/workspace/spec.md` · `features/auth/data-model.md` (F1 precedent) · `00-architecture.md` §5, §8 · `ADR-001` (Prisma + Postgres) · `ADR-002` (shared contracts) · `Implementation Plan.md` F2
**Owner:** `apps/api` — unlike F1, this module's tables are **Prisma-owned** (no library generation). Every model here follows Better Auth's conventions where sensible so the whole schema reads consistently.

---

## 1. Overview

Workspace owns **the container boundary**: one `workspace` row per team, plus the minimal membership row that proves who may cross that boundary (spec rule 5 — *"Membership is the only door"*).

Two tables total:

| Table | Purpose | Formalized by |
|---|---|---|
| `workspace` | The container itself: identity, lifecycle status, name/icon | F2 (this milestone) |
| `workspace_member` | The door: who belongs, with which role | **Seeded in F2**, roles widened by F3 |

The critical structural decision is that **membership exists from day one**. In F2 there are no invitations (F3), so every member is an Owner — but the `requireWorkspaceMember` guard chain that every later milestone copies must run against real membership rows, not against an ad-hoc owner check that F3 would rewrite and backfill. See §3 D2.

All workspace-scoped lifecycle behavior (archive read-only, restore preserves history, permanent delete all-or-nothing) is enforced at the API/service layer on top of these two tables; the data layer provides the state fields, constraints, and cascade rules they need.

---

## 2. Core schema (Prisma-owned)

### 2.1 `workspace`

One row per team container. Server-generated immutable identity; display name is **never** an identifier (spec rules 1–2).

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK `@default(cuid())` | Immutable internal identifier (spec rule 1). Never exposed as sequential/guessable. Inter-table reference key everywhere. |
| `name` | `String` | `@db.VarChar(80)` | Display name only. **No unique constraint** — names may duplicate across workspaces (spec rule 2). Trimmed server-side before persist. |
| `slug` | `String` | `@@unique` | Generated immutable short token used inside URLs `/w/:slug/...` — disambiguates duplicate names in routes without relying on names as identifiers. |
| `status` | `WorkspaceStatus` | `@default(ACTIVE)` | Shared enum: `ACTIVE \| ARCHIVED`. No DELETED value — delete removes the row (§6.3). Drives archive read-only behavior everywhere. |
| `icon` | `String?` | `@db.VarChar(32)` | **Lucide icon key** chosen from the picker (e.g. `"rocket"`, `"boxes"`). Stores the key only — the web app resolves it back to a component through its `IconPair` map. Validated against the shared `WORKSPACE_ICON_KEYS` allow-list (§3 D4). Nullable. No uploads. |
| `archivedAt` | `DateTime?` | — | Set on archive; kept on restore to preserve history; null while active. Read-only enforcement pairs this with `status`. |
| `createdAt` | `DateTime` | `@default(now())` | |
| `updatedAt` | `DateTime` | `@updatedAt` | Renaming never breaks references (spec rule 10) — references use `id`/`slug`, not `name`. |

> No soft-delete flags, no `deletedAt`: permanent delete physically removes the row inside one transaction (§6.3). Archive ≠ delete — archiving is fully represented by `status = ARCHIVED` + `archivedAt`.

```prisma
enum WorkspaceStatus {
  ACTIVE
  ARCHIVED
}

model Workspace {
  id         String          @id @default(cuid())
  name       String          @db.VarChar(80)
  slug       String          @unique
  status     WorkspaceStatus @default(ACTIVE)
  icon       String?         @db.VarChar(32)
  archivedAt DateTime?
  createdAt  DateTime        @default(now())
  updatedAt  DateTime        @updatedAt

  members WorkspaceMember[]

### 2.2 `workspace_member`

One row per user per workspace — the only proof of access (spec rule 5). F2 writes only `role = OWNER`; F3 adds `ADMIN` / `MEMBER` values without any data migration beyond widening the enum.

| Column | Type | Attr | Notes |
|---|---|---|---|
| `id` | `String` | PK `@default(cuid())` | |
| `workspaceId` | `String` | FK → `workspace.id` `onDelete: Cascade` + `@@index([workspaceId])` | Deleting a workspace removes all memberships (spec rule 8). |
| `userId` | `String` | FK → `user.id` `onDelete: Cascade` + `@@index([userId])` | Points at Better Auth's `user` table. Powers "list my workspaces" and the post-auth routing decision from F1. |
| `role` | `WorkspaceRole` | — | Shared enum: `OWNER` in F2; F3 adds `ADMIN`, `MEMBER`. No Owner invitation possible later — F3 constraint, documented here for the trail. |
| `createdAt` | `DateTime` | `@default(now())` | Join date — history preserved through archive/restore (spec rule 9). |

```prisma
enum WorkspaceRole {
  OWNER
  // F3 adds: ADMIN, MEMBER — enum widened in place, existing rows untouched
}

model WorkspaceMember {
  id          String        @id @default(cuid())
  workspaceId String
  userId      String
  role        WorkspaceRole @default(OWNER)
  createdAt   DateTime      @default(now())

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([workspaceId, userId]) // one membership row per user per workspace
  @@index([workspaceId])
  @@index([userId])
  @@map("workspace_member")
}
```

> `user` also gains the back-relation field (`workspaces WorkspaceMember[]`) in its Prisma model block — a schema edit on Better Auth's generated model side is required, documented here so the migration review expects it.

> Cascade direction note: deleting a *membership* row removes only itself; deleting a *workspace* cascades downward into members. User accounts survive workspace deletion (spec rule 8) because no cascade ever points at `user`.

---

## 3. Key decisions & alternatives

### D1 — Ownership lives only in `workspace_member.role` (no `ownerId` column)

**Decision:** The single source of truth for "who owns this workspace" is exactly one `workspace_member` row with `role = OWNER`, enforced by a partial unique index:

```sql
CREATE UNIQUE INDEX workspace_single_owner
  ON workspace_member (workspace_id) WHERE role = 'OWNER';
```

Creation and F3's future ownership transfer insert/swap owner rows within a transaction, so rule 3 (*exactly one Owner*) holds by database constraint, not application discipline.

*Considered and rejected:* a denormalized `workspace.ownerId` column alongside membership (the `Implementation Plan.md` F2 shorthand *"Enforce the `ownerId` and Owner membership invariant"* hints at it). Two representations of one fact can drift under concurrent transfer/removal transactions, and every check would need to remember which to trust. The owner check via membership is one indexed join. **This supersedes the plan's shorthand wording** — sync `Implementation Plan.md` F2 once this doc is locked.

### D2 — Minimal membership table ships in F2, formalized in F3

**Decision:** `workspace_member` is created now with the shape F3 needs, minus extra roles. The guard chain `requireSession → resolve workspaceId → requireWorkspaceMember → requireRole(Owner)` runs against real rows from the first merged route.

*Considered and rejected:* `workspace.ownerId` only, adding membership in F3 — would force F3 to backfill rows from the column mid-project and temporarily ship a `requireWorkspaceMember` that actually checked a single column. Doing it right now is nearly free since F2 creation already must make "creator becomes Owner" atomic (§6.1).

### D3 — `slug` for URLs + `cuid()` for identity; names never identify

Spec rule 1 requires an immutable identifier; rule 2 allows duplicate display names. Routes use a generated immutable `slug`, while inter-table references always use `id`. Alternative (URLs carry raw `cuid()`) satisfies the rules but makes links unwieldy against the switcher UX; cheap now, annoying to retrofit after links exist.

### D4 — Icons are Lucide icon keys, not uploads

The workspace icon picker offers a fixed set of Lucide icons defined client-side as an **`IconPair` map** — `{ [key: string]: LucideIcon }`. On selection, only the **key string** is persisted in `workspace.icon`; rendering resolves the key back through the same map. Rows never hold markup or component data, remain render-framework agnostic, and stay tiny (≤32 chars).

The canonical allowed-key list lives in `packages/shared` as `WORKSPACE_ICON_KEYS` — the single source both apps import: the web builds its `IconPair` from it (every key must map to a real Lucide icon, enforced by types/tests), and the API rejects any icon value outside it. Changing the palette is additive: extend the list, ship new options; old rows keep rendering. Key collisions between similar icons are avoided by naming convention (kebab-case, e.g. `"boxes"`, `"layout-dashboard"`).

R2/upload machinery still belongs to Settings (F11) per `auth/data-model.md §9` if branding ever needs real images. Resolves open question #2 (§10).

---

## 4. Shared contracts (`packages/shared`)

Added in F2, consumed by api and web (ADR-002):

```ts
// zod enums — mirror the Prisma enums above
export const workspaceStatusSchema = z.enum(["ACTIVE", "ARCHIVED"]);
export const workspaceRoleSchema = z.enum(["OWNER"]); // F3 extends

// request/response contracts owned by the workspace module
export const createWorkspaceSchema = z.object({ name: nameSchema, icon: iconSchema.optional() });
export const updateWorkspaceSchema = z.object({ name: nameSchema.optional(), icon: iconSchema.optional() });
export const deleteWorkspaceSchema = z.object({ confirmName: z.string().min(1) }); // exact-name typed confirmation (spec rule 7)

export const workspaceCardSchema = z.object({
  id: z.string(),
  slug: z.string(),
  name: z.string(),
  icon: z.string().nullable(),
  status: workspaceStatusSchema,
  role: workspaceRoleSchema,     // from membership — switcher shows name+icon+role (spec §3.3)
  memberCount: z.number().int(), // meaningful from F3; F2 returns 1
});
export const workspaceDetailSchema = workspaceCardSchema.extend({
  createdAt: z.string().datetime(),
  archivedAt: z.string().datetime().nullable(),
});
```

`nameSchema` trims and enforces the length bound matching `VarChar(80)`; `iconSchema` validates that the value is a member of `WORKSPACE_ICON_KEYS` — the canonical Lucide icon-key list exported from this module and the source behind the web's `IconPair` map (`key → LucideIcon`), so arbitrary strings never enter the DB and both apps resolve keys identically.

---

## 5. Integrity invariants → spec rule mapping

Every behavioral rule in `spec.md §5` lands somewhere concrete:

| Spec rule | Enforcement point |
|---|---|
| 1 — immutable internal id | `id` PK generated once, never re-generated or updated |
| 2 — names may duplicate, never identifiers | no unique index on `name`; routes use `slug`; UI disambiguates with name+icon+role |
| 3 — exactly one Owner | partial unique index `workspace_single_owner` (§3 D1) + transactional seed at creation |
| 4 — all scoped data hangs off one workspace | cascade convention: every future workspace-scoped table carries `workspaceId` FK `Cascade` (§7); `workspace_member` obeys it already |
| 5 — membership is the only door | `requireWorkspaceMember` queries `workspace_member` only; no alternate access path exists |
| 6 — only Owner archives/restores/deletes | `requireRole(Owner)`; OWNER rows guaranteed singular by D1 |
| 7 — delete only from ARCHIVED + exact name typed | service preconditions `status = ARCHIVED` AND `confirmName === name`; enum has no deleted state |
| 8 — delete removes scoped data, never accounts | cascades flow downward only (§2.2 note); user rows structurally unreachable by deletion |
| 9 — archive reversible, restore preserves history | `status` flips, `archivedAt` retained; archive removes nothing |
| 10 — renaming never breaks references | references use `id`/`slug`; `name` purely display |

Integrity summary: two unique constraints (`slug`, `(workspaceId, userId)`), one partial unique (`single_owner`), two FK indexes (`workspaceId`, `userId`).

---

## 6. Lifecycle semantics at the data layer

### 6.1 Creation (atomic, spec §3.1)

Single transaction creates `workspace` + one `workspace_member {role: OWNER}` for the creator. Slug is a random token generated at creation (retry on collision). If either write fails, neither persists — "creator becomes Owner, workspace immediately usable" is one commit.

### 6.2 Archive / restore (reversible, spec §3.2)

Archive sets `status = ARCHIVED` + `archivedAt = now()` and changes nothing else. Restore sets `status = ACTIVE` while **keeping** `archivedAt`, preserving the historical record; memberships, timestamps, and future child data are untouched. Both are Owner-gated and confirmed at the service layer; archived read-only enforcement lives in API/service guards (see `api-design.md`) using these fields as source.

### 6.3 Permanent delete (irreversible, all-or-nothing)

Preconditions (service): workspace is `ARCHIVED` and the request body echoes the exact current name.

Deletion runs in **one transaction**: delete memberships → delete the workspace row → declared `ON DELETE CASCADE` handles all current and future child tables inside the same statement scope. Any failure rolls back completely, leaving the archived workspace unchanged (spec §3.2). No partial states are reachable via constraints alone.

### 6.4 Cross-workspace safety

`requireWorkspaceMember` is the only resolver of workspace context; a non-member receives the same generic rejection whether or not the id/slug exists — no existence leak (F2 done-when).

---

## 7. Cascade deletion contract

How children of `workspace` grow over time — kept here so each future milestone extends rather than re-invents it:

| Table | Introduced | `onDelete` on workspace delete | Notes |
|---|---|---|---|
| `workspace_member` | **F2** | `Cascade` | All memberships die with the workspace (spec rule 8). |
| `invitation` | F3 | `Cascade` | Pending invitations die with the workspace. |
| `project` | F4 | `Cascade` | Project-level issue unassignment logic irrelevant when the whole workspace goes. |
| `issue`, labels, join tables | F5 | `Cascade` | Direct/descendant children. |
| `notification`, `cycle`, `comment` | F6–F8 | `Cascade` | Every workspace-scoped table without exception. |
| `user` (Better Auth) | F1 | **Never touched** | No deletable path points at accounts; accounts outlive workspaces permanently. |

Convention for later milestones: adding any workspace-scoped table **requires** `workspaceId` with `onDelete: Cascade` plus an index — treat a missing cascade as a review defect.

---

## 8. Migration workflow

Unlike F1 (Better Auth CLI generates), workspace models are hand-modeled Prisma:

```bash
# edit apps/api/prisma/schema.prisma per §2 (including User back-relation)
pnpm --filter @shipyard/api db:migrate -- --name add_workspace_and_membership
pnpm --filter @shipyard/api db:generate
```

- Never hand-edit generated migration SQL afterward (plan §5 Step 3).
- The migration produces: 2 tables, 2 enums, the composite/unique/partial indexes above.
- The partial owner index is applied as raw SQL inside the migration or via a paired raw statement in the same migration folder, whichever Prisma's attribute support allows.
- The F1 Testcontainers harness applies migrations automatically each test run — F2 tests inherit real-schema validation.
- Widening `WorkspaceRole` in F3 is an additive enum migration; zero data rewrite.

---

## 9. What we intentionally do NOT model

| Deferred | Why |
|---|---|
| `invitation`, admin/member roles | Owned by Members (F3); table reused, enum widened. |
| Projects/issues/etc. tables | Owned by their features; only the cascade convention (§7) is fixed here. |
| Workspace logo/image uploads | Icons are preset keys (D4); R2 arrives with Settings (F11) if ever needed. |
| Soft delete / trash state | Delete is double-confirmed (archive step + exact-name typing); archive already covers reversible hiding. |
| Workspace count limits | Unlimited per user (open question #1); revisit scale post-MVP. |
| Templates, duplication, orgs, quotas | Spec §6 out of scope. |
| Denormalized `ownerId` column | Rejected by D1; plan wording to be synced after sign-off. |

---

## 10. Open product questions — resolved at data layer

| Spec §7 | Decision |
|---|---|
| 1 — workspace count limit | **No cap.** Unlimited workspaces per user; no DB constraint; revisit scale post-MVP. |
| 2 — icon constraints | **Preset select (Lucide).** The client renders a Lucide-based picker backed by its `IconPair` map (`key → LucideIcon`); selecting saves only the key string (kebab-case, ≤32 chars). `iconSchema` validates against `WORKSPACE_ICON_KEYS` in `packages/shared`, so anything outside the list is rejected before persistence. No uploads, no size/type logic. |
| 3 — onboarding skip | **Sign-in with zero memberships → onboarding flow**, which requests workspace creation before the user can proceed; after creating, they land inside the new workspace. Routing-only concern: zero `workspace_member` rows ⇒ onboard again on next login. No data-layer support needed beyond listing memberships (`GET /api/v1/workspaces`). |

---

## 11. References

- Shipyard: `features/workspace/spec.md`, `features/auth/data-model.md` (conventions precedent), `00-architecture.md`, `ADR-001-stack.md`, `Implementation Plan.md` F2
- Prisma indexes & referential actions: `https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes`
- PostgreSQL partial unique index (single-owner constraint): `https://www.postgresql.org/docs/current/indexes-partial.html`

---

*Next artifact: `api-design.md` — endpoint inventory over these two tables, the canonical guard chain, error codes/envelopes, and app-flow sequence (Next proxy → Express → guard chain → service → repository).*