# ADR-004: Deployment & Infrastructure

- **Status:** Accepted
- **Date:** 2026-08-12

## Context

Shipyard is a portfolio/open-source product: low traffic, no marketing, users self-host. Budget target: **~$0/month** (domain already owned: `yonatanem.com`). The deployment must still demonstrate real production practices (containers, TLS, CI/CD, managed services, backups, observability) — the infrastructure story is part of the portfolio.

## Decision

| Concern | Choice |
|---|---|
| Compute | **Oracle Cloud Always Free** — 1× Ampere A1 VM (2 OCPU / 4GB RAM, Ubuntu 24.04, ~50GB block storage) |
| Runtime | **Docker Compose** on the VPS: `caddy` (TLS, `shipyard.yonatanem.com`), `web` (Next.js :3000), `api` (Express :4000, internal-only) |
| Database | **Neon Postgres (managed)** — IP-allowlisted to the VPS, SSL required; local Postgres container in dev |
| Object storage | **Cloudflare R2** (10GB free, zero egress) — uploads proxied through the API |
| Email | Resend (free tier, custom domain verification) |
| Errors | Sentry (free tier) + Pino structured logs |
| CI/CD | **GitHub Actions** on `main`: lint → typecheck → test → build → push to GHCR → SSH deploy → `prisma migrate deploy` |
| Environments | Local dev + **one production** (no staging for MVP) |
| Self-host story | Repo ships `docker-compose.yml` with **bundled Postgres** for self-hosters; reference deployment uses Neon/R2 (documented in `deployment.md`) |

## Alternatives considered

- **Paid VPS (Hetzner/DigitalOcean ~$5/mo)** — perfectly good, but the goal is $0; Oracle free covers it.
- **Railway / Render / Fly.io** — fast deploys, but paid at this scale and teach less about server administration (the curriculum explicitly covers VPS hosting, Linux, Caddy/Nginx).
- **AWS / Azure free tiers** — 12-month expiry (t2/t3.micro, B1S) would force a migration mid-project; good for future projects.
- **GCP e2-micro** — always-free but 1GB RAM is too tight for web + API + builds.
- **Self-hosted Postgres on the VPS** — rejected: managed Neon gives backups/PITR/SSL out of the box, one less service to operate, and demonstrates the production pattern of separating compute from state.
- **Kubernetes / multiple VMs / load balancers** — grossly premature for this traffic; explicitly deferred.

## Reason

Oracle's free tier is the only genuinely comfortable always-free option (4 cores / 24GB pool — we use a 2 OCPU / 4GB slice, leaving headroom for a future Grafana stack or worker). The split — compute self-hosted, state managed — is how real SaaS is built and is a stronger portfolio statement than "everything on one box." It also maps 1:1 to the curriculum (Docker Compose, VPS, Caddy/SSL, CI/CD, backups, observability).

## Consequences

- **Positive:** ~$0/month infra; production-real architecture; every component is a build-in-public story; self-hosters get a one-command deploy.
- **Negative:** Neon free cold starts (~1–2s after idle; acceptable, optional keep-alive); free-tier limits (0.5GB DB, 10GB R2, 100 emails/day) — ample for MVP traffic; Oracle signup friction / Ampere capacity quirks; single box = single point of failure (acceptable for the MVP, noted in `deployment.md`).
