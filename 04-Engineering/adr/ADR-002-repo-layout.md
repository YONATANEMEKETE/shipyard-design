# ADR-002: Repository Layout — Monorepo

- **Status:** Accepted
- **Date:** 2026-08-12

## Context

Shipyard has three code concerns — Next.js web app, Express API, and the shared contracts between them — plus a separate design/planning repo. The layout must support shared TypeScript/Zod contracts, a single CI pipeline, easy local dev, and an approachable open-source contribution experience.

## Decision

**Single monorepo `shipyard`, pnpm workspaces + Turborepo:**

```
shipyard/
├── apps/
│   ├── web/          Next.js (App Router) — public surface
│   └── api/          Express + TypeScript — modular monolith
├── packages/
│   └── shared/       Zod schemas + shared TS types/enums (API contracts)
├── docker-compose.yml
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```

The planning/design repo (`shipyard-design`) **remains separate** as the product source of truth (docs, UX, UI, architecture, ADRs — this document).

## Alternatives considered

- **Two separate repos** (`shipyard-web`, `shipyard-api`) — clean separation, but: shared Zod contracts force either duplication or a third package anyway; two CI pipelines; two repos to open-source and maintain. Rejected.
- **Single repo without workspaces** — no package isolation, no caching, messy imports. Rejected.

## Reason

One source of truth for contracts (the *Contract Mental Model*: web and API can never drift apart), one CI pipeline (lint → test → build → deploy), Turborepo's task caching and `turbo dev` developer experience, and a single open-source artifact to star/fork — matching the Build Stage goal of a strong public repo. Also exercises the Turborepo/pnpm workspaces backlog item.

## Consequences

- **Positive:** shared validation/type safety end-to-end; contributors run `pnpm install && pnpm dev` and get both apps; design history stays cleanly separated from code.
- **Negative:** web and API version together (fine for a monolith product); Turborepo adds config surface; release automation must respect workspace boundaries.
