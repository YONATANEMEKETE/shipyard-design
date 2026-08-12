# ADR-001: Tech Stack

- **Status:** Accepted
- **Date:** 2026-08-12

## Context

Shipyard needs a stack that: satisfies the PRD (auth + email verification + OAuth, RBAC, issues/projects/cycles with archival, comments/mentions, notifications, search & filters, dashboards); demonstrates production-grade full-stack engineering for the Build Stage portfolio; matches the current learning path (7-phase Express/Node/PostgreSQL curriculum — apply-immediately principle); and stays approachable for open-source contributors (self-host story).

## Decision

| Layer | Choice |
|---|---|
| Frontend | **Next.js (App Router)** + React + shadcn/ui + Tailwind v4 (Harbor Amber theme export) + TanStack Query + Zod |
| Backend | **Express + TypeScript — modular monolith** (`apps/api`) |
| Database | **PostgreSQL** via **Prisma** (Neon managed in prod, local container in dev) |
| Auth | **Better Auth** — email/password + email verification, Google/GitHub OAuth, sessions, password reset |
| Contracts | Zod schemas in `packages/shared` (single source of truth) |
| API style | REST, **`/api/v1` versioned from day one** |
| Search | **Postgres full-text** (`tsvector` + `ts_rank`, GIN index) |
| Notifications | In-app, **light polling (~60s)** — no realtime in MVP |
| Email | Resend |
| Async | **No queues / WebSockets / workers in MVP** (worker slot reserved on the VPS) |

## Alternatives considered

- **NestJS / Hono / Fastify** — stronger structure or lighter weight, but the entire backend curriculum is Express-flavored; adopting it would stall the 80/20 build flow. Deferred to future Build Stage projects.
- **tRPC / GraphQL** — great DX, but REST + OpenAPI matches the resource-oriented PRD, the open-source public-API story (post-MVP), and the curriculum notes (*Resource-oriented API design*, *OpenAPI at Practical Depth*).
- **Drizzle** — viable ORM alternative; Prisma chosen because the curriculum's data layer (SQL → schema design → query patterns) maps to it and it ships migrations/type-safety out of the box.
- **Hand-rolled auth** — tempting after the JWT/session notes, but Better Auth covers the exact PRD auth surface and saves weeks; auth internals remain learnable via the codebase.
- **Meilisearch / Typesense** — explicitly post-MVP per PRD ("advanced search" deferred); Postgres FTS is sufficient at MVP scale.
- **WebSockets / polling-everything-else** — no realtime requirement in the MVP; polling is the minimal correct answer.

## Reason

This stack is the intersection of: PRD coverage, the current learning track (implementing what was just studied = the Build Stage's "learn → apply immediately" rule), shadcn v4 + Tailwind v4 compatibility (theme already exported), and open-source contributor friendliness (one repo, familiar tools, no exotic infra).

## Consequences

- **Positive:** fastest path to a production-grade MVP; every layer maps to curriculum notes (content potential for the build-in-public story); zero-cost managed services fit the free-tier budget.
- **Negative:** Express needs self-discipline to stay modular (module boundaries enforced by convention + review); no NestJS-style DI container; free-tier constraints (Neon cold starts, 0.5GB storage) must be respected.
