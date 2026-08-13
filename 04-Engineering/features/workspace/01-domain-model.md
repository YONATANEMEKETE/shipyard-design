# Workspace — Domain Model

**Module:** `apps/api/src/modules/workspace`
**Status:** Draft v0.1 — 2026-08-12
**PRD source:** §5.2 Workspace · §7.1 Workspace Rules · §5.11 Settings (workspace portions)

---

## 1. Overview & Scope

Workspace owns the **container**: the top-level boundary that isolates one team's data — its lifecycle (active → archived → deleted), its identity rules, and its membership existence.

**In scope:**
- Workspace identity (immutable id vs display name)
- Creation, update (name/icon), archive, restore, permanent deletion
- The one-Owner invariant and the lifecycle guards around it
- Workspace switching data (list for the switcher with name/icon/role context)
- The scope anchor every other module's queries hang off

**Out of scope (owned by other modules):**
- Roles, invitations, membership management, ownership transfer mechanics → `members` module (the Workspace doc only asserts the invariants members must honor — e.g., exactly one Owner, Owner cannot leave before transfer)
- Workspace Settings editing UI, member directory → `settings`
- Projects/issues/cycles content → their own modules (they reference the Workspace id)

**Key separation:** Workspace = the **boundary** (what data belongs where). Members = **who can cross it** and with what role.

---

## 2. Domain Entities

### 2.1 Workspace

The team container. Every project, issue, cycle, membership, and workspace-scoped notification hangs off exactly one Workspace.

| Attribute | Notes |
|---|---|
| `id` | **Immutable internal identifier** — the only identity used for routing, permissions, invitations, and data relationships. Never changes. |
| `name` | Display label only. **May duplicate** other workspaces' names (even among workspaces visible to the same user). Never used as an identifier. |
| `icon` | Optional (R2 object URL) — purely cosmetic, helps distinguish duplicate names in the switcher. |
| `ownerId` | Exactly one Owner per workspace; the creator becomes the Owner. |
| `lifecycle state` | `Active` · `Archived` · (deleted = row removed). See §3. |

**Invariants:**
- The internal identifier is immutable, unique, and never user-supplied (server-generated).
- Renaming a workspace changes nothing but the display label — no reference breaks, no identifier changes.
- Every member belongs to exactly one Workspace per membership; a user may hold memberships in many workspaces, with **roles assigned independently per workspace**.

### 2.2 Membership (reference — owned by `members` module)

Workspace existence implies memberships: a user either is a member of a Workspace or has no access at all. The Workspace domain asserts the invariants members must enforce:

- Every workspace has exactly one Owner.
- The Owner cannot leave or be removed until ownership is transferred (transfer mechanics: `members`).
- Project ownership follows membership changes (owned projects transfer to the Workspace Owner) — enforced in `members`.

---

## 3. Lifecycle State Machine

```
                 ┌──────────┐
        create ─▶│  ACTIVE  │◀───────────────┐
                 └────┬─────┘                │ restore
                      │ archive (Owner)      │
                 ┌────▼─────┐                │
                 │ ARCHIVED │────────────────┘
                 │ read-only│
                 └────┬─────┘
                      │ permanent delete (Owner, exact-name typed)
                 ┌────▼─────┐
                 │ DELETED  │  (irreversible — row + all scoped data removed)
                 └──────────┘
```

| Transition | Guard | Effect |
|---|---|---|
| create | authenticated user | Creator = Owner; workspace immediately usable |
| **archive** | Owner only, confirm | Becomes read-only; no longer actively usable; restorable |
| **restore** | Owner only, confirm | Back to Active; data + history preserved |
| **permanent delete** | Owner only; workspace must be Archived; **exact workspace name typed** | Removes workspace + all scoped resources, settings, memberships, invitations — **user accounts untouched**; irreversible |

**Rules:**
- Archive → delete is the only deletion path (an Active workspace cannot be deleted directly).
- Failed deletion rolls back atomically — the Archived workspace remains unchanged.
- Deletion does not delete member user accounts or their data in other workspaces.
- A user accessing an archived workspace by direct URL gets the read-only archived summary (Owner) or is denied (non-owner).

---

## 4. Domain Invariants

From PRD §7.1 (Workspace Rules) + §5.2, condensed:

1. Every workspace has exactly one immutable, unique internal identifier.
2. Workspace names may duplicate and are never used as identifiers (identity = id, always).
3. Every workspace has exactly one Owner.
4. A user may belong to multiple workspaces; roles are assigned independently per workspace.
5. All projects, issues, cycles, and members belong to exactly one workspace.
6. Users can only access workspaces they are members of (membership = the only door).
7. An active workspace must be archived before permanent deletion.
8. Only the Owner can archive, restore, or permanently delete a workspace.
9. Permanent deletion requires the exact workspace name typed by the Owner (confirmation gate).
10. Deleting a workspace removes all workspace-scoped data and memberships **without deleting user accounts**.
11. Workspace deletion is atomic: it succeeds completely or not at all; a failed deletion leaves the Archived workspace unchanged.
12. Renaming a workspace does not change its identifier or break existing references.
13. Archived workspaces are read-only; restoration preserves data and history.

---

## 5. Domain Operations

| Operation | Description | Requires |
|---|---|---|
| `createWorkspace` | New workspace; creator becomes Owner; immediately active | verified user |
| `getWorkspaces` | All workspaces the user belongs to (for switcher + auto-enter) | verified user |
| `getWorkspace` | Single workspace scoped by membership (active or archived) | member |
| `updateWorkspace` | Rename / change icon | Owner |
| `archiveWorkspace` | Active → Archived (read-only) | Owner + confirm |
| `restoreWorkspace` | Archived → Active | Owner + confirm |
| `deleteWorkspace` | Archived → permanently removed (all scoped data) | Owner + exact name |
| `getArchivedWorkspaces` | Owner's archived list (switcher section) | Owner |
| `resolveActiveWorkspace` | Auto-enter logic: sole workspace → dashboard; many → selection | verified user |

*Ownership transfer (`transferOwnership`) is documented in `members` — the Workspace domain only asserts the post-transfer invariant: exactly one Owner remains, transferring Owner becomes Admin.*

---

## 6. Relationships to Other Modules

```
                    ┌─────────────┐
                    │  WORKSPACE  │  ← container + lifecycle (this module)
                    └──────┬──────┘
       ┌───────────┬───────┼────────┬──────────────┐
       ▼           ▼       ▼        ▼              ▼
   members      projects  cycles   issues      settings
   (who can     (scoped   (scoped  (scoped     (workspace
    enter)       by wsId)  by wsId) by wsId)    settings UI)
       │
       └── auth: verified identity provides the user;
           workspace membership provides the access context
```

- **Scoping anchor:** every query in every module is filtered by `workspaceId` — resolved from the session + route, never from client input (architecture §7.4).
- **Auth:** workspace flows require a *verified* user (auth module); membership/roles decide access (members module).
- **Settings:** workspace settings UI (details, members tab, danger zone) reads/writes through this module's operations + members.
- **Deletion cascade:** workspace delete invokes each owning module's cleanup (projects, issues, cycles, memberships) inside one transaction — orchestrated by the workspace service.

---

## 7. Trust Boundaries & Security Properties

1. The workspace identifier is server-generated and immutable; client-supplied names are never used for identity or routing.
2. Membership is the only access path: every operation starts with `requireWorkspaceMember` (resolved from session userId + workspaceId in the route).
3. Owner-only actions (archive/restore/delete/rename) additionally pass `requireRole(Owner)`.
4. Duplicate names are safe by design — switching/selection always uses the id; the UI shows name + icon + role to disambiguate.
5. Archived workspaces are read-only at the data layer (write operations rejected), not just hidden in the UI.
6. Permanent deletion is the most destructive operation in the system: Owner-only + archived-only + exact-name typing + atomic transaction.

---

## 8. Non-Goals (MVP)

Per PRD §5.2 future enhancements: workspace templates, duplication, custom branding, organization-level management. Also excluded: cross-workspace features, workspace limits/quotas, workspace analytics.

---

## 9. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Workspace count limit per user? | PRD has none — default unlimited |
| 2 | Icon upload constraints (type/size)? | Follow the uploads-through-API pattern (R2) — decide in data model |
| 3 | Archived workspace URL access for non-owners | Redirect to active workspace + message (decide in API design) |
| 4 | Onboarding "skip" behavior | Next login returns to onboarding until a workspace exists (PRD flow 3) — confirm no escape hatch needed |
