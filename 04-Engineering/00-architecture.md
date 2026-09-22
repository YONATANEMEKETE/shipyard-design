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
| `adr/ADR-003-web-api-communication.md` | Next.js proxy, internal API, no CORS | ⛔ superseded by ADR-006 |
| `adr/ADR-004-deployment-infra.md` | Oracle VPS, Neon, R2, Caddy, CI/CD | ⛔ superseded by ADR-007 |
| `adr/ADR-005-mcp-server-surface.md` | MCP server surface & hosting (endpoint in the API, workspace-bound tokens, JSON responses) | ✅ done — amended by ADR-006 |
| `adr/ADR-006-public-web-api-split.md` | Public API origin, direct browser calls, CORS allowlist, shared session cookie | ✅ done |
| `adr/ADR-007-hosting-vercel-render.md` | Vercel (web) + Render (API); Neon/R2/Resend kept; platform deploys | ✅ done |
| `features/*` | Per-feature behavior specs (`spec.md`); technical design produced at each feature's implementation step | ✅ F1–F11 implemented · F13 (MCP) designed, implementation next |
| `deployment.md` | Topology, env matrix, deploy flow, migrations, DNS, smoke test, runbook | ✅ done |

**Reading order:** 00-architecture → ADRs → features (behavior specs) → deployment. Per-feature technical design is produced during each feature's implementation step (Implementation Plan §5, Step 2).

---

## 2. System Context

```text
Browser ──HTTPS──▶ shipyard.yonatanem.com        (Vercel — Next.js web)
   │                    │
   │                    └─ server-side session check ──▶ API
   │
   └─ cross-origin fetch, credentials ──▶ api.shipyard.yonatanem.com
Agent ──HTTPS /mcp─────────────────────▶ (Render — Express API, public)
                                           │
                                           ├─▶ Neon Postgres
                                           ├─▶ Cloudflare R2
                                           ├─▶ Resend
                                           └─▶ Sentry

   CI/CD: GitHub Actions quality gate + platform git deploys (push to main)
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
8. **Two origins, one product.** The web app (`shipyard.yonatanem.com`, Vercel) and the API (`api.shipyard.yonatanem.com`, Render) are separate public origins of the same site (ADR-006); the API is public by design — every route authenticates on its own.

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
  → TanStack Query (client component) — direct fetch to the API origin
  → https://api.shipyard.yonatanem.com/api/v1/issues  (cross-origin, credentials + CORS)
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
- The browser talks to the API directly, cross-origin; the API trusts only its own authentication (session cookie or bearer token) and validates every input regardless of caller.

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
| **Security headers** | Helmet on Express; exact-origin CORS with credentials (ADR-006); CSP via Next (planned) |
| **Health & shutdown** | `/healthz` + `/readyz` endpoints; graceful shutdown (drain connections, close DB) |
| **Idempotency** | Required where PRD demands atomicity (ownership transfer, project delete + unassignment); duplicate-submission guards on creation flows |
| **File uploads** | Through the API: validate type/size → upload to R2 server-side → store URL (presigned uploads deferred to post-MVP attachments) |

---

## 9. Data Layer Strategy

- **ORM:** Prisma; schema owned per module but defined in one Prisma schema file (or split schema files, decided at implementation).
- **Database:** Neon Postgres in production (managed — backups, PITR, SSL); local Postgres container in dev (Docker Compose); Neon branch optional for dev DB parity.
- **Migrations:** `prisma migrate` — generated in dev, applied by the API container's start command on deploy (`prisma migrate deploy`), never hand-edited (ADR-007; Render free has no pre-deploy step).
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
- **Reserved slot:** a Render background worker (or scheduled job) for future background work (email digests, cleanup jobs, later integrations). Outbox pattern is the post-MVP path if async side-effects grow.

---

## 12. Deployment & Infrastructure Summary

See `deployment.md` and ADR-004 for detail. Summary:

- **Web:** Vercel (Hobby) — Next.js from the monorepo, `shipyard.yonatanem.com`; `NEXT_PUBLIC_API_URL` is its only API configuration.
- **API:** Render (Free web service, Docker) — `api.shipyard.yonatanem.com`, health check `/healthz`; public by design (session cookies + bearer tokens, ADR-006).
- **Managed services:** Neon (Postgres, SSL required; no IP allowlist on the free plan) · Cloudflare R2 (10GB free, zero egress) · Resend (100 emails/day free; HTTPS API) · Sentry (free tier).
- **CI/CD:** GitHub Actions is the quality gate (lint → typecheck → format → test → build); deployment is the platforms' git integrations on `main`. Migrations run from the API container's start command.
- **Environments:** local dev (compose with local Postgres; the web app on :3000 calls the API on :4000 cross-origin) + one production. No staging for MVP.
- **Backups:** Neon managed backups/PITR; R2 data is re-uploadable (avatars/logos only).
- **Monitoring:** health checks + Pino logs + Sentry now; Grafana/Loki/Prometheus remains post-MVP hardening.

---

## 13. Security Model

- Two public origins (web + API, ADR-006); the API authenticates every route itself (session cookie or bearer token) and is rate-limited per IP and per token.
- Session-based auth via Better Auth (HttpOnly cookies, shared across subdomains in production); CSRF protections per PRD (origin checks against the trusted-origin allowlist).
- RBAC enforced server-side on every route (PRD permission matrix).
- Workspace isolation: every query scoped by workspace membership; cross-workspace access returns 403/404.
- Secrets: only in server environment (`.env` on host / GitHub secrets); never bundled into the client bundle.
- Managed DB locked down: SSL required + long credentials held only in platform env (Neon free has no IP allowlist — recorded deviation in ADR-007).
- Uploads: type/size validation server-side; stored in the public-read R2 bucket under unguessable keys (avatars and future public assets only — private objects need a separate bucket).
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
| 7 | Web ↔ API | Public API origin; browser calls it directly (CORS allowlist, credentials, shared cookie) | ADR-006 |
| 8 | API versioning | `/api/v1` from day one | ADR-001 |
| 9 | Deployment | Vercel (web) + Render (API) + GH Actions quality gate + platform deploys | ADR-007 |
| 10 | Database hosting | Neon managed (prod), local Postgres (dev) | ADR-004 · retained by ADR-007 |
| 11 | Object storage | Cloudflare R2, uploads through API | ADR-004 · retained by ADR-007 |
| 12 | Search | Postgres full-text (`tsvector` + `ts_rank`) | ADR-001 |
| 13 | Notifications | Light polling (~60s), no realtime in MVP | ADR-001 |
| 14 | Email | Resend (verification, invites, password reset) | ADR-001 |
| 15 | Observability | Pino + Sentry (Grafana stack post-MVP) | ADR-004 |
| 16 | Environments | Local dev + single production | ADR-004 |
| 17 | MCP server surface | Endpoint in `apps/api`, reachable at the API origin (`/mcp`); workspace-bound personal access tokens (OAuth later); JSON responses in v1 | ADR-005 · amended by ADR-006 |

---

## 15. Open Questions & Risks

| Item | Notes |
|---|---|
| Neon free cold starts | First query after idle adds ~1 s; acceptable; the API's own cold start dominates (Render free spins down after 15 idle minutes) |
| Free-tier limits | Neon 0.5 GB / 100 CU-hrs, R2 10 GB, Resend 100 emails/day, Render 750 instance-hrs (one always-awake service ≈ 744) — ample for MVP traffic; revisit at scale |
| Mobile app (future) | The API is public (ADR-006), so a mobile client can call it directly; its origin joins the CORS/trusted-origin allowlist when that project starts |
| Contributors/self-hosters | Repo ships `docker-compose.yml` with bundled Postgres for self-hosters; reference deployment uses Neon/R2 (documented in `deployment.md`) |
| Search quality | English-only stemming acceptable for MVP; Meilisearch is the post-MVP upgrade path |
