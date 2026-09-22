# ADR-007: Hosting — Vercel (web) + Render (API)

- **Status:** Accepted
- **Date:** 2026-09-22
- **Supersedes:** ADR-004 (Oracle VPS + Docker Compose + Caddy)
- **Depends on:** ADR-006 (the public web/API split, which removed the one assumption that required a shared network)

## Context

ADR-004 chose Oracle Cloud Always Free + Docker Compose + Caddy for a $0/month, production-real deployment. In practice the Oracle path never became dependable: signup capacity errors, a silently halved Always Free Ampere allowance, and instance-level reclamation risk meant provisioning and keeping the VM was a recurring cost of attention — for a deployment that is a portfolio/demo surface, not a service with an operator. Meanwhile the codebase's shape (Next.js + Express, stateless API, managed state) matches two managed platforms whose free tiers cover this scale.

## Decision

| Concern | Choice |
|---|---|
| Web | **Vercel (Hobby)** — Next.js from the `shipyard` repo, root directory `apps/web` |
| API | **Render (Free web service, Docker)** — the API Dockerfile; health check `/healthz`; domain `api.shipyard.yonatanem.com` |
| Database | **Neon Postgres (managed)** — unchanged from ADR-004 |
| Object storage | **Cloudflare R2** — unchanged (public-read bucket; domain `assets.yonatanem.com`) |
| Email | **Resend** — unchanged (HTTPS API; Render's free tier blocks SMTP ports 25/465/587, which this stack does not use) |
| Errors | **Sentry** + Pino structured logs — unchanged |
| CI/CD | **Platform git integrations** — push to `main` deploys both; `.github/workflows/ci.yml` remains the quality gate (lint → typecheck → format → test → build). GHCR, SSH deploy and Caddy are retired |
| Migrations | `prisma migrate deploy` from the API container's start command (Render free has no pre-deploy step and no shell) |
| Environments | Local dev + one production. No staging for the MVP |
| Self-host story | Kept: the repo ships the dev `docker-compose.yml` (bundled Postgres) and the API Dockerfile; the reference deployment uses Neon/R2/Resend |

Free-tier facts the plan depends on (a snapshot — re-verify before planning against them):

- Render free web services **spin down after 15 idle minutes**; wake takes ~1 minute; 750 instance-hours/month per workspace (one always-awake service ≈ 744 hours, so a ~14-minute keep-warm ping fits). No shell, no pre-deploy commands, rollback to the last two deploys only.
- **Render free Postgres is deleted 30 days after creation** — not used; Neon remains the database.
- **Vercel Hobby is non-commercial** by its terms; Shipyard is MIT open source with no revenue. Revisit (Pro) if that changes.
- Neon free: 0.5 GB storage, 100 CU-hours/month per project, scale-to-zero after 5 minutes (~1 s first-query wake), 6-hour restore history.

## Alternatives considered

- **Fix Oracle instead** — the friction is structural (signup capacity, allowance changes, reclamation), not a one-off; every fix keeps the box and its operations surface. Rejected.
- **Railway** — pleasant DX, but there is no meaningful free tier: the Free plan is $1/month of usage credit, and a 0.5 GB always-on service alone costs ≈$5/month, so the real price is Hobby at $5/month. Rejected on cost; the API Dockerfile stays portable to it.
- **Fly.io / Hetzner / Render Starter ($7/month)** — the "always-on, no cold start" options at ~$3–7/month. Retained as the documented upgrade path: Render's instance type flips in one click and the same Dockerfile runs on any of them.
- **Render for both web and API** — one dashboard, but the web app loses the Next.js platform integration (build pipeline, image optimization, edge middleware) for no gain. Rejected.
- **Keep the single-box topology of ADR-004** — superseded by ADR-006; there is no managed network spanning Vercel and Render.

## Reason

The deployment is a portfolio surface: real (TLS, CI, managed state, observability) without being a second job. Two managed platforms with git integrations remove the box, the proxy and the deploy pipeline in one move, while keeping every managed service that already worked. The $0 target is preserved; the honest price is cold starts on the free API tier and the free-tier caveats above.

## Consequences

- **Positive:** no server to administer; deploys are pushes; the split (ADR-006) becomes natural; fewer moving parts (no Caddy, no production compose, no GHCR, no SSH); Neon/R2/Resend/Sentry carry over unchanged; the self-hosting story stays intact.
- **Negative / to manage:** API cold starts (~1 min after 15 idle minutes; a keep-warm ping every ~14 min fits inside the 750 instance hours, and the web middleware's degrade path covers the blip); migrations run from the container start command (idempotent, but ADR-004's "migrations on the host, one credential home" separation is gone); Render may restart free services at any time (graceful shutdown is implemented); Vercel Hobby's non-commercial terms.
- **Revisit when:** cold starts or free-tier limits hurt real usage (upgrade Render to Starter), a second environment becomes necessary, or traffic outgrows Neon's free compute.

## Deviations from ADR-004 (recorded, not silent)

- **Neon IP allowlist dropped** — IP Allow is a Scale-plan feature. Compensating control: SSL required, long credentials, and the connection string living only in the platform environment.
- **Migrations in the container start command** instead of a pre-deploy step (pre-deploy commands are paid-only on Render free).
- **`TRUST_PROXY_HOPS=1` remains correct** — Render's edge is the one proxy hop, exactly as Caddy was before it.
