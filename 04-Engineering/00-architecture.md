# Shipyard — System Architecture

**Status:** Draft v0.1 — approved for engineering documentation
**Date:** 2026-08-12
**Applies to:** `shipyard` monorepo (web + api + shared)

---

## 1. Purpose & Document Map

This document is the **map of the entire Shipyard system**. It defines the high-level architecture that every other engineering document builds on. Per-feature technical design (domain model, data model, API design) is intentionally **not** pre-written here — it is produced during each feature's implementation step, driven by the behavioral feature spec (`features/*/spec.md`).

| Document | Contents | Status |
|---|---|---|
| `00-architecture.md` | This document — system context, principles, modules, request lifecycle | ✅ done |
| `adr/ADR-001-stack.md` | Tech stack decisions (Next.js, Express, Prisma, Better Auth, Zod) | ✅ done |
| `adr/ADR-002-repo-layout.md` | Monorepo layout, pnpm workspaces, Turborepo, shared contracts | ✅ done |
| `adr/ADR-003-web-api-communication.md` | Next.js proxy, internal API, no CORS | ✅ done |
| `adr/ADR-004-deployment-infra.md` | Oracle VPS, Neon, R2, Caddy, CI/CD | ✅ done |
| `features/*` | Per-feature behavior specs (`spec.md`); technical design produced at each feature's implementation step | ⏳ next |
| `deployment.md` | Compose layout, CI/CD pipeline, backups, observability runbook | ⏳ planned |

**Reading order:** 00-architecture → ADRs → features (behavior specs) → deployment. Per-feature technical design is produced during each feature's implementation step (Implementation Plan §5, Step 2).

---

## 2. System Context

```
                        ┌──────────────────────────────────────────────┐
                        │              Oracle VPS (Ampere)              │
   Browser              │  ┌──────────┐   ┌──────────────────────────┐  │
     │   HTTPS          │  │  Caddy   │──▶│  Docker Compose network  │  │
     ▼                  │  │  :80/443 │   │  ┌────────────────────┐  │  │
  ┌────────┐            │  │ (TLS)    │   │  │ Next.js web :3000  │  │  │
  │  User  │            │  └──────────┘   │  │  (public surface)  │  │  │
  └────────┘            │                 │  └─────────┬──────────┘  │  │
     │                  │                 │            │ proxied     │  │
     ▼                  │                 │  ┌─────────▼──────────┐  │  │
  shipyard.yonatanem.com│                 │  │ Express API :4000  │  │  │
                        │                 │  │ (internal only)    │  │  │
                        │                 │  └─────────┬──────────┘  │  │
                        │                 │            │ (future:    │  │
                        │                 │            │  worker)    │  │
                        └─────────────────┼────────────┼─────────────┘
                                          │            │
                     ┌────────────────────┼────────────┼─────────────────────┐
                     │  Managed services  │            │                     │
                     │  ┌──────────────┐  │   ┌────────▼─────────┐  ┌───────▼──┐
                     │  │ Neon Postgres │◀─┘   │ Cloudflare R2    │  │  Resend  │
                     │  │ (managed DB)  │      │ (avatars, logos) │  │ (email)  │
                     │  └──────────────┘      └──────────────────┘  └──────────┘
                     │  Google OAuth ──┐       GitHub OAuth ──┐      Sentry (errors)
                     └────────────────┴──────────────────────┴──────────────────┘

   CI/CD: GitHub Actions ── lint → typecheck → test → build → push GHCR → SSH deploy
```

**External services:** Neon Postgres (prod DB) · Cloudflare R2 (object storage) · Resend (transactional email) · Google & GitHub OAuth (Better Auth) · Sentry (error tracking).

---

## 3. Architecture Principles

1. **Modular monolith.** One deployable API with explicit feature modules and a shared kernel. No microservices.
2. **Backend owns state and rules.** All business rules, invariants, and permissions live in the API; the database is the source of truth; the client is never trusted.
3. **Workspace isolation on every query.** Every data access is scoped by the authenticated user's workspace — enforced in services, not in the UI.
4. **No artificial complexity.** Technologies exist because the product needs them (PRD rule). No queues, WebSockets, or search engines in the MVP.
5. **Archive ≠ delete.** Archived resources are read-only and reversible; permanent deletion is a separate, confirmed, atomic operation.
6. **Contracts are shared.** Zod schemas in `packages/shared` are the single source of truth between web and API (and future mobile).
7. **Validation at the edge.** All input is validated at the API boundary before any business logic runs.
8. **One public surface.** Caddy → Next.js is the only externally reachable entry point in production.

---

## 4. Module Map

The API is a modular monolith. Each module owns its domain; the shared kernel provides infrastructure.

```
apps/api/src/
├── modules/
│   ├── auth/           Better Auth integration: email/password, verification,
│   │                   Google/GitHub OAuth, sessions, password reset
│   ├── users/          User profiles, account settings, theme preference
│   ├── workspace/      Workspace CRUD, archive/restore/delete, ownership,
│   │                   workspace switching data
│   ├── members/        Invitations, roles (Owner/Admin/Member), membership
│   │                   management, ownership transfer
│   ├── issues/         Issue CRUD, workflow statuses, blocked flag, labels,
│   │                   assignees, due dates, archive, list/kanban data
│   ├── projects/       Project CRUD, ownership, progress, archive, delete+
│   │                   unassignment (atomic)
│   ├── cycles/         Cycle lifecycle (start/complete/reopen/archive),
│   │                   no-overlap + one-active rules
│   ├── comments/       Issue comments, mentions, edit history
│   ├── notifications/  Assignment + mention notifications, read/unread,
│   │                   unread-count polling endpoint
│   ├── search/         Global search (issues/projects/cycles/members),
│   │                   filters, sorting, saved views
│   ├── settings/       Account + workspace settings endpoints
│   └── dashboard/      Aggregates: my issues, active projects, current
│                       cycle, recent activity
└── shared/             Prisma client, config (env), errors, logger (Pino),
    │                   middleware (request-id, rate-limit, security),
    │                   permissions (RBAC helpers), health checks
```

**Frontend (apps/web):** Next.js App Router with route groups mirroring the modules:
`(auth)`, `(workspace)/dashboard`, `(workspace)/issues`, `(workspace)/projects`, `(workspace)/cycles`, `(workspace)/members`, `settings`. Auth pages are custom, following the approved design (`03-UI` of the design repo).

**Shared package (packages/shared):** Zod schemas for every API contract (request/response), shared enums (statuses, priorities, roles), and generated types consumed by both web and API.

---

## 5. Module Dependency Rules

- Feature modules depend **only** on the shared kernel — never on each other's internals.
- Cross-module operations go through the owning module's service (e.g., `notifications` calls `issues`/`members` services to build notifications; `dashboard` reads other modules' query services).
- **Projects and Cycles are independent entities** — they never reference each other; any project↔cycle relationship is derived through issues (PRD rule).
- No circular imports. A module may read another module's *data* only via its public service API.
- The shared kernel is dependency-free of feature modules (kernel may import Prisma/models only).

---

## 6. Layered Architecture (per module)

Every module follows the same internal layering:

```
Route (HTTP mapping)
  → Validation (Zod, "validation at the edge")
  → Permission check (RBAC + workspace scoping)
  → Controller (request/response shaping, status codes)
  → Service (business rules, invariants, transactions)
  → Repository (Prisma data access)
  → Database (Neon / local Postgres)
```

- **Route:** thin mapping of method + path to handler (`/api/v1/issues`).
- **Validation:** Zod schema from `packages/shared` — rejects bad input before anything else runs.
- **Permission check:** shared RBAC helper using the PRD permission matrix + workspace membership; runs before the service.
- **Controller:** orchestrates the request; no business logic.
- **Service:** owns business rules and invariants (e.g., cycle no-overlap, blocked-clears-on-Done, project deletion + unassignment atomicity); begins transactions.
- **Repository:** Prisma queries; returns domain-shaped data; never exposes raw DB errors.

---

## 7. Request Lifecycle

### 7.1 Read path (e.g., `GET /api/v1/issues?status=IN_PROGRESS`)

```
Browser
  → TanStack Query (client component) or server component fetch
  → Next.js route handler / server fetch
  → http://api:4000/api/v1/issues   (internal Docker network, cookie forwarded)
  → request-id middleware (assigns + logs request id)
  → Pino structured log (method, path, request-id, user, duration)
  → Better Auth session check (authn)         [401 if invalid]
  → workspace context resolution (from session)
  → Zod query validation                       [400 if invalid]
  → permission check (workspace member)        [403 if denied]
  → issues service (filters, full-text search, sorting)
  → Prisma repository (workspace-scoped query)
  → Neon
  ← JSON response (envelope consistent with error shape)
  ← Next.js renders / forwards to client
```

### 7.2 Write path (e.g., `POST /api/v1/projects`, transfer ownership)

Same path up to the service, then:

```
  → service begins Prisma transaction
  → business rules validated inside transaction
  → side effects in same transaction (e.g., project ownership transfer,
    notification creation for assignment/mention)
  → commit / rollback (atomic — partial failure never persists)
  → 201/200 response with created/updated resource
```

### 7.3 Error handling

- Centralized error handler maps domain errors → consistent JSON shape:
  `{ error: { code, message, details? } }` with correct 4xx/5xx status.
- Zod errors → 400 with field details. Permission failures → 403. Unknown errors → 500 (logged + Sentry capture, generic message to client).
- Failed drag/status updates return the previous state + error (PRD).

### 7.4 Trust boundaries

- All incoming data is untrusted until Zod validation (per `Backend System Mental Model`).
- Workspace/resource ids come from the session context, never trusted from the client body.
- The browser never talks to the API directly; it only sees Next.js responses.

---

## 8. Cross-Cutting Concerns

| Concern | Approach |
|---|---|
| **Authentication** | Better Auth (Express server adapter) — email/password + email verification, Google/GitHub OAuth, sessions, password reset. Custom auth UI on Next side |
| **Authorization** | Shared permissions layer implementing the PRD RBAC matrix (Owner/Admin/Member); enforced per route + per resource |
| **Validation** | Zod schemas from `packages/shared` at every API boundary |
| **Config** | 12-factor: `env` handling in shared kernel; validated at boot (fail fast); `.env.example` committed; secrets only in server env |
| **Logging** | Pino structured logs with request-id correlation |
| **Error tracking** | Sentry (API + web); captures only unexpected errors |
| **Rate limiting** | Per-IP limits on auth endpoints (login, register, resend) + global API limits |
| **Security headers** | Caddy + Helmet-equivalent on Express; strict CORS disabled (no cross-origin browser calls); CSP via Next |
| **Health & shutdown** | `/healthz` + `/readyz` endpoints; graceful shutdown (drain connections, close DB) |
| **Idempotency** | Required where PRD demands atomicity (ownership transfer, project delete + unassignment); duplicate-submission guards on creation flows |
| **File uploads** | Through the API: validate type/size → upload to R2 server-side → store URL (presigned uploads deferred to post-MVP attachments) |

---

## 9. Data Layer Strategy

- **ORM:** Prisma; schema owned per module but defined in one Prisma schema file (or split schema files, decided at implementation).
- **Database:** Neon Postgres in production (managed — backups, PITR, SSL); local Postgres container in dev (Docker Compose); Neon branch optional for dev DB parity.
- **Migrations:** `prisma migrate` — generated in dev, applied in CI before deploy (`prisma migrate deploy`), never hand-edited.
- **Full-text search:** `tsvector` generated columns + GIN index; `ts_rank` ordering; English config (post-MVP: Meilisearch).
- **Archival pattern:** archived resources carry archived state + timestamp; read-only; restoration returns to stored pre-archive state (PRD).
- **Transactions:** all multi-step writes (ownership transfer, project deletion + unassignment, notification side-effects) run in single Prisma transactions.

---

## 10. Frontend Architecture

- **Next.js App Router**; route groups mirror modules; loading/error/empty states follow the design system (empty states are permission-aware).
- **Server components** for initial data; **client components** for interactivity (kanban drag-drop, modals, forms, tabs).
- **TanStack Query** for client-side data fetching, caching, mutations, and the **~60s notification polling** (unread-count endpoint).
- **shadcn/ui + Tailwind v4** consuming the exported theme (`03-UI/exports/globals.css` — Harbor Amber, light + dark).
- **View preferences** (list/kanban per user per workspace) stored server-side per PRD.
- **Auth pages** custom-built from the design repo (Auth Shell, signup/login/verification screens).

---

## 11. Async Stance

- **No queues, no WebSockets, no background workers in the MVP.**
- Notifications are created synchronously inside the owning transaction (assignment/mention).
- Client refreshes via polling; no push.
- **Reserved slot:** a `worker` container on the same VPS compose file for future background work (email digests, cleanup jobs, later integrations). Outbox pattern (per curriculum) is the post-MVP path if async side-effects grow.

---

## 12. Deployment & Infrastructure Summary

See `deployment.md` and ADR-004 for detail. Summary:

- **Host:** Oracle Cloud Always Free — 1× Ampere A1 VM (2 OCPU / 4GB RAM), Ubuntu 24.04, ~50GB block storage.
- **Runtime:** Docker Compose — `caddy` (TLS, `shipyard.yonatanem.com`), `web` (Next.js :3000), `api` (Express :4000, internal only); Postgres **not** on the VPS.
- **Managed services:** Neon (Postgres, IP-allowlisted to VPS, SSL required) · Cloudflare R2 (10GB free, zero egress) · Resend (100 emails/day free) · Sentry (free tier).
- **CI/CD:** GitHub Actions on `main`: lint → typecheck → test → build → push image to GHCR → SSH deploy → `prisma migrate deploy`.
- **Environments:** local dev (compose with local Postgres) + one production. No staging for MVP.
- **Backups:** Neon managed backups/PITR; R2 data is re-uploadable (avatars/logos only).
- **Monitoring:** health checks + Pino logs + Sentry now; Grafana + Loki + Prometheus as post-MVP hardening on the same box.

---

## 13. Security Model

- Single public entry point (Caddy → Next); API port never published to the host.
- Session-based auth via Better Auth (HttpOnly cookies); CSRF protections per PRD.
- RBAC enforced server-side on every route (PRD permission matrix).
- Workspace isolation: every query scoped by workspace membership; cross-workspace access returns 403/404.
- Secrets: only in server environment (`.env` on host / GitHub secrets); never bundled into the client bundle.
- Managed DB locked down: SSL required + IP allowlist (VPS public IP).
- Uploads: type/size validation server-side; stored under private R2 bucket, served via signed URLs.
- Rate limits on auth + global API; security headers; input validation at every boundary.

---

## 14. Decision Log

| # | Decision | Choice | ADR |
|---|---|---|---|
| 1 | Frontend framework | Next.js (App Router) | ADR-001 |
| 2 | Backend | Express + TypeScript, modular monolith | ADR-001 |
| 3 | ORM / DB | Prisma + PostgreSQL (Neon managed / local) | ADR-001 |
| 4 | Auth | Better Auth (email/password, OAuth, sessions) | ADR-001 |
| 5 | Validation/contracts | Zod in `packages/shared` | ADR-001 |
| 6 | Repo layout | Monorepo: pnpm workspaces + Turborepo (`apps/web`, `apps/api`, `packages/shared`) | ADR-002 |
| 7 | Web ↔ API | Next.js proxies; API internal-only (no CORS) | ADR-003 |
| 8 | API versioning | `/api/v1` from day one | ADR-001 |
| 9 | Deployment | Oracle VPS + Docker Compose + Caddy + GH Actions | ADR-004 |
| 10 | Database hosting | Neon managed (prod), local Postgres (dev) | ADR-004 |
| 11 | Object storage | Cloudflare R2, uploads through API | ADR-004 |
| 12 | Search | Postgres full-text (`tsvector` + `ts_rank`) | ADR-001 |
| 13 | Notifications | Light polling (~60s), no realtime in MVP | ADR-001 |
| 14 | Email | Resend (verification, invites, password reset) | ADR-001 |
| 15 | Observability | Pino + Sentry (Grafana stack post-MVP) | ADR-004 |
| 16 | Environments | Local dev + single production | ADR-004 |

---

## 15. Open Questions & Risks

| Item | Notes |
|---|---|
| Neon free cold starts | First request after idle adds ~1–2s; acceptable; optional keep-alive ping from VPS cron |
| Free-tier limits | Neon 0.5GB, R2 10GB, Resend 100 emails/day — ample for MVP traffic; revisit at scale |
| Mobile app (future) | API stays internal; future mobile reachability decided when that project starts |
| Contributors/self-hosters | Repo ships `docker-compose.yml` with bundled Postgres for self-hosters; reference deployment uses Neon/R2 (documented in `deployment.md`) |
| Search quality | English-only stemming acceptable for MVP; Meilisearch is the post-MVP upgrade path |
