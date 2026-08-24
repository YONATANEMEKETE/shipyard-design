# Workspace — Feature Spec

**Status:** Approved
**Last updated:** 2026-08-22
**Design sources:** PRD §5.2 · §7.1 · UX User Flows 3–4
**Technical design:** Excluded by design — produced during this feature's implementation step, driven by this behavioral spec.

---

## 1. What this feature is about

Workspace is the **container**: the top-level boundary that isolates one team's data and lifecycle (active → archived → deleted). Every project, issue, cycle, and membership hangs off exactly one workspace. Workspace = the boundary; Members = who can cross it and with what role; Auth = who is verified.

## 2. What users can do

- Create a workspace (creator becomes Owner).
- See the workspaces they belong to and switch between them.
- Rename a workspace and change its icon.
- Archive a workspace (Owner) and restore it (Owner).
- Permanently delete an archived workspace (Owner, typing the exact name).
- Enter a workspace from onboarding, invitation, or the workspace switcher.

## 3. Main behaviors & actions

### 3.1 Creation & identity
- Workspace identity (id) is server-generated and immutable; **the display name is never an identifier** — names may even duplicate between workspaces.
- Creator becomes the workspace Owner; workspace is immediately usable.
- A user may belong to many workspaces.

### 3.2 Lifecycle
- **Archive** (Owner, confirmed): workspace becomes read-only; no longer actively usable; fully restorable.
- **Restore** (Owner, confirmed): back to active; data and history preserved.
- **Permanent delete** (Owner only): the only path is archived → delete; requires the exact workspace name typed; removes the workspace and **all** workspace-scoped data, memberships and invitations — user accounts are untouched. Irreversible.
- Deleting succeeds completely or not at all; a failed deletion leaves the archived workspace unchanged.
- Direct URL access to an archived workspace: owners see the read-only archived view; non-owners are denied.

### 3.3 Switching & entry
- After login, the user is routed to: pending invitation → onboarding (no workspace) → dashboard (one workspace) → workspace selection (multiple).
- The switcher shows name + icon + role to disambiguate duplicate names.

## 4. User flows (high level)

1. **Onboarding:** first login → invite or create workspace → creator = Owner → land in workspace.
2. **Switching:** switcher → pick workspace → navigate with workspace context.
3. **Archive:** settings → danger zone → confirm → read-only archived workspace.
4. **Delete:** archive first → danger zone → type exact name → permanent removal of all scoped data.

## 5. Business rules

1. Every workspace has exactly one immutable internal identifier.
2. Workspace names may duplicate and are never identifiers.
3. Every workspace has exactly one Owner (Members enforces the invariant on transfer).
4. All projects, issues, cycles, memberships, and workspace-scoped notifications belong to exactly one workspace.
5. Membership is the only door: users can only access workspaces they are members of.
6. Only the Owner can archive, restore, or permanently delete.
7. Permanent deletion requires the archived state + exact name typed.
8. Deleting a workspace removes all workspace-scoped data but never user accounts.
9. Archiving is reversible; restoration preserves data and history.
10. Renaming never breaks references.

## 6. Out of scope (MVP)

Workspace templates, duplication, custom branding, org-level management, cross-workspace features, quotas, analytics.

## 7. Open product questions

| # | Question | Notes |
|---|---|---|
| 1 | Workspace count limit per user | None in PRD — unlimited |
| 2 | Icon upload constraints | Type/size; decide at implementation |
| 3 | Onboarding "skip" behavior | Next login returns to onboarding until a workspace exists |
