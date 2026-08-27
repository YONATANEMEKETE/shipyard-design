# Shipyard Implementation Plan

**Status:** Ready for implementation  
**Application repository:** [`shipyard`](https://github.com/YONATANEMEKETE/shipyard)  
**Planning repository:** [`shipyard-design`](https://github.com/YONATANEMEKETE/shipyard-design)  
**Scope:** MVP implementation sequence

This document is the ordered implementation plan for Shipyard. It converts the feature specifications into implementable milestones named `F1`, `F2`, `F3`, and so on.

Each feature specification is **behavior-only**:

```text
04-Engineering/features/<feature>/spec.md
```

Technical details (data model, API design, system design) are deliberately **not** pre-written. They are planned per feature at the start of its implementation milestone (see §5, Step 2), driven by the behavioral spec — so the plan stays relevant at implementation time instead of being second-guessed.

This plan answers a different question:

> In what order should the modules be implemented so that each feature has the dependencies, contracts, data, and UI it needs?

---

## 1. Implementation principles

### 1.1 Implement vertical slices

A feature is not complete when only its API exists. Each feature should be implemented as a vertical slice across the monorepo:

```text
packages/shared
    ↓
apps/api
    ↓
apps/web
    ↓
tests and verification
```

A normal feature should include:

- Shared request and response schemas in `packages/shared`
- Prisma schema changes and a migration
- API module implementation
- Permission and workspace-scope enforcement
- API route and service tests
- Web data access and mutation logic
- Loading, error, empty, and permission-aware states
- UI implementation against the approved design
- Documentation updates where an architectural decision changes

### 1.2 The backend owns rules

The API is the source of truth for:

- Business rules
- Validation
- Permissions
- Workspace isolation
- State transitions
- Transactions
- Archival and deletion behavior

The web application must not reproduce business rules as its authority. Client-side checks are for user experience only; the API always checks again.

### 1.3 Use the existing module boundaries

The API is a modular monolith. Feature modules must not import each other's internals.

```text
Route
  → validation
  → permission check
  → controller
  → service
  → repository
  → Prisma
```

Cross-module work goes through a public service contract owned by the target module.

### 1.4 Keep workspace isolation on every query

Every workspace-scoped request must follow the standard guard chain:

```text
requireSession
  → resolve workspaceId from the URL
  → requireWorkspaceMember
  → requireRole where needed
  → workspace-scoped service and repository queries
```

The workspace ID is a lookup key, not proof of access.

### 1.5 Archive is not delete

Archived resources are read-only, reversible, and excluded from normal active lists. Permanent deletion is a separate confirmed operation with an explicit transaction and cascade contract.

### 1.6 Complete one milestone before starting its dependent milestone

A milestone can have internal checkpoints, but it should not be marked complete until its acceptance criteria and integration tests pass.

---

## 2. Foundation status — F0

### F0 — Repository and engineering foundation

**Status:** Complete

The repository foundation is already implemented in the `shipyard` repository:

- pnpm workspace
- Turborepo task graph
- `apps/web`
- `apps/api`
- `packages/shared`
- Node and pnpm pinning
- pnpm-only installation enforcement
- ESLint
- TypeScript checks
- Prettier
- EditorConfig and VS Code configuration
- Environment-file protection
- Husky and Commitlint
- GitHub collaboration files
- GitHub Actions quality workflow
- README and CONTRIBUTING documentation
- Protected `main` branch

The foundation quality gate is:

```bash
pnpm check
```

The current CI gate is:

```text
repository policy
  → pnpm install --frozen-lockfile
  → pnpm audit --audit-level=high
  → pnpm lint
  → pnpm typecheck
  → pnpm format:check
  → pnpm build
```

F0 is not a product feature. It is the prerequisite for all product milestones below.

---

## 3. Dependency graph and implementation order

The high-level order is:

```text
F1 Auth and Identity
        ↓
F2 Workspace Lifecycle
        ↓
F3 Members and RBAC (core)
        ↓
F4 Projects
        ↓
F3 Members and RBAC (integration completion)
        ↓
F5 Issues and Labels
        ↓
F6 Notifications
        ↓
F7 Cycles
        ↓
F8 Comments and Mentions
        ↓
F9 Dashboard
        ↓
F10 Search
        ↓
F11 Settings and Preferences
        ↓
F12 MVP Hardening and Release Readiness
```

Some dependencies are intentionally completed through integration checkpoints:

- Members need the Projects service to transfer owned projects during removal or leave.
- Issues need the Cycles relation only after the Cycle module exists.
- Issues emit assignment notifications, so Notifications completes the issue integration.
- Comments emit mention notifications, so Comments follows Notifications.
- Settings is last because its specification deliberately delegates to Auth, Workspace, Members, and the owning resource modules.

### Milestone summary

| ID | Feature | Main dependency | Status |
|---|---|---|---|
| F0 | Repository and engineering foundation | None | ✅ complete |
| F1 | Auth and identity | F0 | ✅ complete |
| F2 | Workspace lifecycle | F1 | ⏳ next |
| F3 | Members, invitations, and RBAC | F1, F2, F4 integration | planned |
| F4 | Projects | F1, F2, F3 core | planned |
| F5 | Issues and labels | F1, F2, F3, F4 | planned |
| F6 | Notifications | F5 | planned |
| F7 | Cycles | F5, F6 contracts | planned |
| F8 | Comments and mentions | F5, F6, F3 | planned |
| F9 | Dashboard | F4, F5, F7 | planned |
| F10 | Search | F3, F4, F5, F7 | planned |
| F11 | Settings and preferences | F1–F10 ownership contracts | planned |
| F12 | MVP hardening and release readiness | F1–F11 | planned |

---

## 4. Feature milestones

## F1 — Auth and identity

**Specification:** [`features/auth/spec.md`](../features/auth/spec.md)  
**Depends on:** F0  
**Unblocks:** every authenticated feature

### Scope

Implement the account and session foundation using Better Auth as decided in ADR-001.

### Backend work

- Configure Better Auth with the Express server adapter.
- Implement email/password registration and login.
- Implement email verification and verification resend.
- Implement Google and GitHub OAuth.
- Implement logout, session lookup, password reset, password change, and email change.
- Mount the Better Auth handler under `/api/v1/auth`.
- Configure HttpOnly session cookies.
- Add session middleware consumed by all other modules.
- Add authentication rate limits.
- Add generic responses where the PRD forbids account-existence leaks.
- Add the auth data model and migration for User, Account, Session, and Verification.
- **Tests:** write route and service tests with the prepared harness (see `testing-infrastructure-plan.md`): Supertest against `app.ts` via Vitest with a Testcontainers Postgres; the first migration lands in `prisma/migrations` and is applied automatically by `vitest.global-setup.ts` on every test run.

### Shared and web work

- Add auth request and response contracts where custom wrapping is needed.
- Add the Next.js proxy behavior from ADR-003.
- Build the approved custom auth screens:
  - Sign up
  - Sign in
  - Email verification
  - Verification pending
  - Forgot password
  - Reset password
  - OAuth error
- Implement the post-auth routing decision:
  - Pending invitation
  - No workspace → onboarding
  - One workspace → dashboard
  - Multiple workspaces → workspace selection

### Done when

- A user can register, verify, log in, log out, and restore a session.
- OAuth success and failure flows work for both configured providers.
- Password reset tokens are single-use and expiring.
- Sessions are HttpOnly and are not stored in client JavaScript.
- Protected API routes return the correct unauthenticated response.
- Auth service, route, and critical flow tests pass.
- A clean browser session can complete the designed auth journey.

---

## F2 — Workspace lifecycle

**Specification:** [`features/workspace/spec.md`](../features/workspace/spec.md)  
**Depends on:** F1  
**Unblocks:** every workspace-scoped feature

### Scope

Implement workspace creation, selection, switching, archive, restore, and deletion while establishing the URL-based workspace context defined in the specification.

### Backend work

- Add the Workspace data model and migration.
- Implement create, list, get, update, archive, restore, and delete operations.
- Create the workspace and initial owner relationship atomically.
- Implement `workspaceId` route context resolution.
- Implement the shared `requireWorkspaceMember` guard chain.
- Enforce the `ownerId` and Owner membership invariant.
- Enforce archived workspace read-only behavior.
- Implement confirmed archive and permanent deletion behavior.
- Add workspace-scoped error codes and response envelopes.

### Shared and web work

- Add workspace contracts and shared status/role enums.
- Build onboarding workspace creation.
- Build workspace selection and switching.
- Add workspace context to the application shell.
- Add workspace archive and restore screens.
- Add the owner-only danger-zone deletion flow.
- Ensure the active workspace lives in the URL, never hidden server state.

### Done when

- A verified user can create a workspace and becomes its Owner.
- The workspace list supports zero, one, and multiple workspace states.
- Every workspace-scoped request checks membership.
- Archived workspaces are read-only and restorable.
- Permanent deletion follows the documented cascade contract.
- Cross-workspace access is rejected without leaking resource existence.
- Workspace onboarding and switching work through the approved flows.

---

## F3 — Members, invitations, and RBAC

**Specification:** [`features/members/spec.md`](../features/members/spec.md)  
**Depends on:** F1 and F2  
**Integration dependency:** F4 provides project ownership transfer

### Scope

Implement workspace membership, invitations, roles, permission checks, ownership transfer, and member lifecycle operations.

### Checkpoint A — Core membership

Implement first:

- Membership data model and migration.
- Invitation data model and partial unique constraints.
- Member listing and member details.
- Owner, Admin, and Member role checks.
- Invite, resend, revoke, accept, and decline flows.
- Immediate permission changes on the next request.
- Ownership transfer between current members.
- Invitation email adapter through Resend, with a local development mode.

### Checkpoint B — Project integration

After F4 exposes `projectsService.transferOwnedProjects(...)`, complete:

- Remove member transaction.
- Leave workspace transaction.
- Automatic transfer of owned projects, including archived projects.
- Owner-leave protection and transfer requirements.
- Full rollback behavior if the membership or ownership operation fails.

### Shared and web work

- Add role, invitation, membership, and permission contracts.
- Implement permission-aware navigation and empty states.
- Build member directory, invitations, role management, remove, leave, and transfer screens.
- Show the project transfer count in removal confirmation.

### Done when

- Invitations are race-safe and idempotent.
- An invitation cannot grant Owner role.
- Role changes apply immediately.
- Removed members lose access on their next request.
- Admin permissions follow the PRD matrix exactly.
- Member removal and leave atomically transfer project ownership.
- Every permission failure is tested at the API boundary.

---

## F4 — Projects

**Specification:** [`features/projects/spec.md`](../features/projects/spec.md)  
**Depends on:** F1, F2, and F3 core membership

### Scope

Implement project CRUD, status lifecycle, ownership, progress contract, archive/restore, and atomic deletion behavior.

### Backend work

- Add the Project data model and migration.
- Implement create, list, get, update, archive, restore, and delete.
- Enforce normalized workspace-scoped project-name uniqueness.
- Implement project ownership transfer with in-transaction membership revalidation.
- Expose `projectsService.transferOwnedProjects(...)` for Members.
- Implement project archive and restore semantics.
- Implement atomic deletion with issue unassignment through `SetNull`.
- Define progress and issue-summary query contracts for later Issues integration.
- Add project activity records where required by the feature specification.

### Shared and web work

- Add project schemas, status enums, and response cards.
- Build projects list, create modal, project detail, edit, archive, restore, and delete flows.
- Build project ownership transfer UI.
- Leave issue-based progress empty or explicitly pending until F5 is integrated; do not fake progress in the client.

### Done when

- Owners/Admins can manage projects according to the permission matrix.
- Members can read and perform permitted edits.
- Project names are normalized and race-safe.
- Archive and restore are reversible.
- Deleting a project atomically unassigns its issues without deleting them.
- Member removal can call the project ownership transfer service.

---

## F5 — Issues and labels

**Specification:** [`features/issues/spec.md`](../features/issues/spec.md)  
**Depends on:** F1, F2, F3, and F4

### Scope

Implement the core product workflow: issue creation, identifiers, list/board views, status changes, labels, assignment, archiving, and deletion.

The Cycle relation is added in the F7 integration checkpoint; F5 should not create a fake cycle implementation.

### Backend work

- Add Issue, Label, IssueLabel, and activity data models.
- Implement workspace-scoped issue identifier allocation such as `SHIP-024`.
- Implement create, list, get, update, archive, restore, and delete.
- Implement statuses, priorities, blocked state, due dates, assignee, and project assignment.
- Implement normalized label uniqueness and label management.
- Implement list and Kanban query shapes in one server-side query path.
- Implement workspace-scoped validation for projects, assignees, and labels.
- Enforce archived issue read-only behavior.
- Define the internal assignment-notification contract for F6.
- Keep all multi-write operations transactional.

### Shared and web work

- Add IssueCard, IssueDetail, Label, status, priority, and filter contracts.
- Build issue list, Kanban board, create/edit dialog, issue detail, labels, archive, restore, and delete flows.
- Implement drag-and-drop status changes with recovery on failure.
- Add permission-aware actions and empty states.
- Use the stored view preference when F11 is available; default safely before then.

### Done when

- Users can create and manage issues within their workspace.
- Issue identifiers are unique and never reused.
- List and Kanban views use the same authoritative issue data.
- Status and blocked invariants are enforced by the API.
- Assigning or reassigning an issue exposes the notification event contract.
- Project deletion leaves issues alive with cleared project assignments.
- Issue CRUD, status transitions, label operations, and workspace isolation are tested.

---

## F6 — Notifications

**Specification:** [`features/notifications/spec.md`](../features/notifications/spec.md)  
**Depends on:** F5 for issue references and assignment events

### Scope

Implement in-app notifications as synchronous transaction side effects. The MVP does not use WebSockets, queues, or workers.

### Backend work

- Add the Notification data model and migration.
- Implement the internal `notificationsService.create(...)` contract.
- Integrate assignment notifications into issue create and reassignment transactions.
- Implement unread count, list, mark read, mark all read, delete, and clear-all endpoints.
- Enforce recipient ownership on every notification operation.
- Add indexes for the polling hot path.

### Shared and web work

- Add notification type and response contracts.
- Build the header bell, unread badge, notification panel, read state, and clear actions.
- Configure TanStack Query polling at approximately 60 seconds.
- Ensure notification navigation handles archived issues and deleted references correctly.

### Done when

- Assignment notifications are created only when the assignee actually changes.
- Notifications are created in the same transaction as the source action.
- The browser can poll unread count without WebSockets.
- Users cannot read or delete another user's notification.
- Notification behavior is tested for duplicate and cascade cases.

---

## F7 — Cycles

**Specification:** [`features/cycles/spec.md`](../features/cycles/spec.md)  
**Depends on:** F5 issue foundation and F6 shared notification contracts

### Scope

Implement cycle scheduling, lifecycle transitions, no-overlap rules, one-active-cycle rule, and issue-cycle assignment.

### Backend work

- Add Cycle data model and migration.
- Add the Postgres exclusion constraint for date overlap.
- Add the partial unique index for one active cycle.
- Implement create, list, get, update, start, complete, reopen, archive, restore, and delete.
- Implement controlled transitions instead of a generic status update.
- Add the nullable Issue-to-Cycle relation and migration.
- Integrate cycle validation into issue create and update.
- Implement cycle progress through issue queries, not duplicated counters.

### Shared and web work

- Add cycle contracts and lifecycle action schemas.
- Build cycle list, detail, create/edit, start, complete, reopen, archive, restore, and delete flows.
- Build cycle issue views by reusing the issue endpoint with `cycleId` filters.
- Show conflict details for overlap and active-cycle failures.

### Done when

- Concurrent cycle writes cannot violate overlap or active-cycle constraints.
- Invalid lifecycle transitions return documented errors.
- Archived cycles restore to their stored previous status when valid.
- Issues can be assigned, filtered, and displayed by cycle.
- Cycle progress is derived from issues.
- Cycle and issue integration tests pass.

---

## F8 — Comments and mentions

**Specification:** [`features/comments/spec.md`](../features/comments/spec.md)  
**Depends on:** F3 members, F5 issues, and F6 notifications

### Scope

Implement issue comments, author-only editing/deletion, mention parsing, and mention notifications.

### Backend work

- Add Comment data model and migration.
- Implement chronological comment listing with cursor pagination.
- Implement create, edit, and delete with authorship checks.
- Reject new comments on archived issues.
- Parse and resolve mentions against current workspace members.
- Deduplicate mentions per comment.
- Call Notifications inside the same comment transaction.
- Do not re-notify on comment edits.

### Shared and web work

- Add comment and mention contracts.
- Build the issue detail comments section.
- Build the comment composer and member mention suggestions.
- Build inline edit, delete confirmation, edited indicator, and empty states.
- Link mention and notification navigation to issue details.

### Done when

- Only comment authors can edit or delete comments.
- Mention suggestions use current workspace members.
- Duplicate mentions create one notification per member.
- Archived issues reject new comments.
- Comment and notification writes are atomic.
- Comment pagination and author permission behavior are tested.

---

## F9 — Dashboard

**Specification:** [`features/dashboard/spec.md`](../features/dashboard/spec.md)  
**Depends on:** F4 projects, F5 issues, and F7 cycles

### Scope

Implement the composed workspace dashboard as a read-only aggregation surface.

### Backend work

- Implement the dashboard query service and endpoint.
- Aggregate My Work, active projects, current cycle, blocked issues, overdue issues, and recent activity.
- Reuse owning module query contracts instead of duplicating domain rules.
- Implement issue recently-viewed recording as the documented internal side effect.
- Keep the dashboard load-on-navigation; no polling is required.

### Shared and web work

- Add dashboard aggregate contracts.
- Build dashboard cards, empty states, loading states, and permission-aware sections.
- Link cards to issue, project, cycle, and workspace routes.
- Match the approved dashboard UI and responsive behavior.

### Done when

- One dashboard request returns the composed payload.
- Empty states render correctly when there is no current cycle, active project, or assigned work.
- Dashboard data is workspace-scoped.
- No N+1 query pattern is introduced.
- Recently viewed behavior matches the specification.

---

## F10 — Search

**Specification:** [`features/search/spec.md`](../features/search/spec.md)  
**Depends on:** F3 members, F4 projects, F5 issues, and F7 cycles

### Scope

Implement workspace-scoped global search using PostgreSQL full-text search and the documented bounded result shapes.

### Backend work

- Add the Postgres search migration with generated `tsvector` columns and GIN indexes.
- Implement the grouped search endpoint for issues, projects, cycles, and members.
- Implement type-ahead suggestions using the same bounded endpoint.
- Add filtering, sorting, and result limits from the API specification.
- Enforce workspace isolation on every result type.
- Use owning module card contracts; do not create parallel domain shapes.

### Shared and web work

- Add search query, filter, grouped-result, and suggestion contracts.
- Build global search, debounce behavior, search-within filters, result grouping, and keyboard navigation.
- Add loading, no-result, invalid-query, and permission-aware states.

### Done when

- Search never returns data from another workspace.
- Full-text ranking and bounded limits work on realistic seed data.
- Search-as-you-type does not create unbounded server work.
- Result cards reuse issue, project, cycle, and member contracts.
- Search indexes and query behavior are covered by integration tests.

---

## F11 — Settings and preferences

**Specification:** [`features/settings/spec.md`](../features/settings/spec.md)  
**Depends on:** F1–F10 ownership and guard contracts

### Scope

Implement the settings surface last because Settings delegates domain ownership to Auth, Workspace, Members, and the owning resource modules.

### Backend work

- Implement profile reads and updates through the settings service.
- Implement appearance and theme preference.
- Implement avatar upload validation and R2 adapter integration.
- Implement workspace-scoped view preference reads and upserts.
- Delegate password, email, workspace, and membership operations to their owning modules.
- Enforce the correct user-scoped and workspace-scoped guards.

### Shared and web work

- Add profile, appearance, avatar, and view preference contracts.
- Build account, appearance, workspace, members, and danger-zone settings screens.
- Integrate issue and project list/Kanban preferences into their owning pages.
- Add upload loading, failure, size/type validation, and signed-URL handling.

### Done when

- Settings does not duplicate Auth, Workspace, or Members business logic.
- User and workspace settings use the correct scope.
- Theme and view preferences persist across sessions.
- Avatar uploads follow the R2 security contract.
- The settings shell is permission-aware and matches the approved UI.

---

## F12 — MVP hardening and release readiness

**Depends on:** F1–F11

This is the final MVP readiness milestone rather than a product module.

### Backend and data hardening

- Complete unit, API integration, and database integration coverage for critical invariants.
- Verify every workspace-scoped query and permission matrix entry.
- Add rate limits for auth, writes, comments, invitations, and polling endpoints.
- Add centralized error mapping and structured Pino logging.
- Add request IDs, health checks, readiness checks, and graceful shutdown.
- Verify Prisma migration and rollback procedures.
- Verify archive, restore, cascade, and transaction behavior.
- Add Sentry error capture for unexpected failures.

### Frontend hardening

- Verify every screen has loading, empty, error, and permission-aware states.
- Verify failed drag/status mutations restore the previous UI state.
- Verify keyboard navigation, focus management, responsive behavior, and dark mode.
- Verify auth redirects, workspace switching, deep links, and browser refresh behavior.

### Deployment and operations

- Add the Docker Compose development and self-host configuration.
- Add web and API production Dockerfiles.
- Add Caddy configuration and internal-only API networking.
- Add GHCR image publishing to CI.
- Add SSH deployment to the Oracle VPS.
- Run `prisma migrate deploy` during deployment.
- Configure Neon, R2, Resend, OAuth, and Sentry secrets outside the repository.
- Add backup, restore, health, and incident runbook documentation.

### Release gate

The MVP is ready for release only when:

```bash
pnpm install --frozen-lockfile
pnpm check
pnpm test
pnpm build
```

and the deployment smoke test passes against the production-like environment.

---

## 5. Standard implementation loop for every feature

Use this loop for every `F1`–`F11` milestone.

### Step 1 — Read the feature spec

Read the behavioral feature spec:

```text
04-Engineering/features/<feature>/spec.md
```

It defines what the feature is about, what users can do, the main behaviors, and the business rules. Check the related PRD, UX flows, screen inventory, and UI design before writing code.

### Step 2 — Plan the feature's technical design

This is where the technical details are decided — per feature, at implementation time, driven by the spec:

- Confirm dependencies and contracts with existing modules (which service calls which, who owns each table and invariant).
- Produce the feature's technical design: domain model, data model, API design, and system-design decisions.
- Record decisions that change the high-level architecture as ADRs in the shipyard repository.
- Add or update shared Zod schemas and enums.
- Resolve the spec's open product questions, or record them as explicitly deferred.
- Keep the design artifacts with the implementation (shipyard repo docs), so code and design live together.

The behavioral spec is the requirements source; the technical design serves it — never the other way around.

### Step 3 — Implement the data layer

- Update the Prisma schema according to the feature data model.
- Add the migration through Prisma.
- Add indexes and database constraints.
- Add repository methods.
- Add transaction tests for multi-write behavior.

Never hand-edit generated migrations after Prisma creates them.

### Step 4 — Implement the API module

Follow the module structure:

```text
apps/api/src/modules/<feature>/
├── routes.ts
├── controller.ts
├── service.ts
├── repository.ts
└── schemas.ts
```

The exact file split can evolve, but the boundary must remain explicit:

```text
route → validation → permission → controller → service → repository
```

### Step 5 — Implement the web slice

- Add the route or route group.
- Add server-side initial data where appropriate.
- Add TanStack Query hooks for client interactions.
- Add forms and mutations using shared schemas.
- Add loading, error, empty, and permission-aware states.
- Match the approved Harbor Amber design system.

### Step 6 — Test the feature

At minimum, test:

- Happy path
- Invalid input
- Unauthenticated access
- Non-member access
- Incorrect role
- Cross-workspace IDs
- Archived resource writes
- Concurrent or atomic behavior where specified
- Delete and cascade behavior
- Web loading, error, and empty states

### Step 7 — Run the repository gate

```bash
pnpm check
```

Once the test runner exists:

```bash
pnpm test
```

Then inspect the final diff and update the relevant engineering documentation if implementation decisions changed.

---

## 6. Branch and commit convention for feature milestones

Create one branch per focused feature or feature slice:

```bash
git switch main
git pull --ff-only origin main
git switch -c feature/f1-auth
```

Use smaller branches when a feature is large:

```text
feature/f1-auth-api
feature/f1-auth-ui
feature/f1-auth-tests
```

Use Conventional Commits:

```text
feat(auth): add email registration
feat(workspace): add workspace creation transaction
feat(issues): add issue status transitions
test(auth): cover expired reset tokens
fix(members): prevent admin ownership transfer
docs(engineering): record implementation decision
```

Before opening a pull request:

```bash
pnpm check
```

If `main` advances:

```bash
git fetch origin
git rebase origin/main
git push --force-with-lease
```

Merge with the repository's rebase strategy after the `quality` check passes.

---

## 7. Explicitly deferred from the MVP

The following are not part of the feature sequence unless the product plan changes:

- WebSockets and realtime updates
- Queues and background workers
- Notification push delivery
- Public browser-to-API access and CORS
- Mobile application
- Meilisearch or another external search engine
- Presigned attachment uploads
- Advanced reporting
- Billing and usage limits
- Multi-region deployment
- Kubernetes and microservices

The MVP remains a modular monolith with synchronous transactions, polling for notifications, PostgreSQL full-text search, and one public Next.js surface.

---

## 8. Immediate next milestone

The most recently completed milestone is:

```text
F1 — Auth and identity
```

F1 delivered registration, email verification, password reset and change, Google and GitHub OAuth, Better Auth sessions mounted under `/api/v1/auth`, session middleware consumed by other modules, and the post-auth routing decision (pending invitation → onboarding → dashboard → workspace selection).

The next implementation milestone is:

```text
F2 — Workspace lifecycle
```

Before starting F2, prepare:

- Workspace data model and migration plan
- URL-based workspace context resolution design
- Shared `requireWorkspaceMember` guard chain approach
- Owner membership invariant enforcement plan
- Archive/restore semantics and confirmed deletion cascade contract
- Workspace-scoped error codes and response envelope conventions
- Onboarding creation, selection, switching, archive/restore, and danger-zone deletion screen checklist

F2 is complete only when a verified user can create a workspace and become its Owner, move between zero, one, and multiple workspace states, switch workspaces through the approved flows, and every workspace-scoped request validates membership without leaking cross-workspace resource existence.
