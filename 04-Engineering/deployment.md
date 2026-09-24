# Deployment

**Status:** Draft v1 — written against ADR-006 and ADR-007
**Last updated:** 2026-09-24
**Sources:** `adr/ADR-006-public-web-api-split.md` · `adr/ADR-007-hosting-vercel-render.md` · `adr/ADR-005-mcp-server-surface.md` · `00-architecture.md` §12
**Owner:** `shipyard` repo — operational document; every value here must match the running system.

> **Locked scope (2026-09-22):** $0/month stack — Vercel Hobby (web), Render free Docker web service (API), Neon Postgres, Cloudflare R2, Resend, Sentry. One production environment plus local dev. No staging.

## 1. Topology

```text
Browser ──▶ https://shipyard.yonatanem.com        Vercel · Next.js (apps/web)
   │   fetch + credentials (cross-origin, exact-origin CORS)
   └─────▶ https://api.shipyard.yonatanem.com     Render · Express (apps/api)
Agent ────▶ https://api.shipyard.yonatanem.com/mcp
                        │
                        ├─▶ Neon Postgres (SSL required)
                        ├─▶ Cloudflare R2 (avatars; public-read, unguessable keys)
                        ├─▶ Resend (verification, reset, invites)
                        ├─▶ Sentry (unexpected errors)
                        └─▶ Grafana Cloud (traces, metrics, logs over OTLP)
```

- Two public origins under one registrable domain (`yonatanem.com`) → same-site, cross-origin: `SameSite=Lax` session cookies keep working.
- The web app proxies nothing. Every relative `/api/v1/…` path resolves against `NEXT_PUBLIC_API_URL` in one shared helper.
- The API's base URL is its own public origin (`API_URL`) — Better Auth's `baseURL`, OAuth redirect URIs and reset links are built from it.

## 2. Environments

| Environment | Web | API | Database |
|---|---|---|---|
| Local dev | `pnpm --filter @shipyard/web dev` → :3000 | `pnpm --filter @shipyard/api dev` → :4000 | docker compose (`pnpm db:up`, host port 5433) |
| Production | Vercel (git integration on `main`) | Render (git integration on `main`) | Neon |

Local dev needs no API-origin configuration — both sides default to `localhost:4000` / `localhost:3000`. Cookies work across the two ports (cookies ignore ports) and `COOKIE_DOMAIN` stays empty.

## 3. Environment matrix

### API — Render

| Variable | Production value | Notes |
|---|---|---|
| `NODE_ENV` | `production` | secure cookies; real email delivery |
| `API_URL` | `https://api.shipyard.yonatanem.com` | Better Auth `baseURL`; OAuth redirect URIs; reset links |
| `WEB_URL` | `https://shipyard.yonatanem.com` | trusted browser origin; verification-email links |
| `COOKIE_DOMAIN` | `yonatanem.com` | shares the session cookie across subdomains (ADR-006) |
| `EXTRA_TRUSTED_ORIGINS` | *(empty; add preview origins as needed)* | extra CORS + trusted origins, comma-separated |
| `TRUST_PROXY_HOPS` | `1` | Render's edge is the one proxy hop |
| `DATABASE_URL` | Neon pooled connection string | SSL required |
| `BETTER_AUTH_SECRET` | 32+ random characters | rotate by redeploy |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Google OAuth app | see §6 |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | GitHub OAuth app | see §6 |
| `RESEND_API_KEY` / `RESEND_FROM` | Resend | HTTPS API (Render free blocks SMTP ports) |
| `R2_ENDPOINT` · `R2_PUBLIC_BUCKET` · `R2_ACCESS_KEY_ID` · `R2_SECRET_ACCESS_KEY` | Cloudflare R2 | `shipyard-bucket`, public-read by design |
| `R2_PUBLIC_BASE_URL` | `https://assets.yonatanem.com` | custom domain, not the r2.dev dev URL |
| `LOG_LEVEL` | `info` | |
| `API_RATE_LIMIT_*` · `AUTH_RATE_LIMIT_*` · `MCP_RATE_LIMIT_*` | defaults | tune against real traffic |
| `SENTRY_API_DSN` | shipyard-api project DSN | error monitoring. Write-only (safe to expose) but per-project, so the value lives in Render, never the repo |
| `SENTRY_RELEASE` | *(empty)* | Render injects `RENDER_GIT_COMMIT` at runtime; set only on hosts that expose no commit SHA |
| `POSTHOG_PROJECT_TOKEN` | PostHog project token (`phc_…`) | product analytics. Write-only (safe to expose) but per-product, so the value lives in Render, never the repo. Empty = the reporter stays off |
| `POSTHOG_HOST` | `https://us.i.posthog.com` | the **ingestion** origin — not the dashboard URL (`us.posthog.com`) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `https://otlp-gateway-prod-eu-west-2.grafana.net/otlp` | Grafana Cloud OTLP gateway. Unset disables all server telemetry (tests, self-hosters) |
| `OTEL_EXPORTER_OTLP_HEADERS` | `Authorization=Basic <base64 instanceID:token>` | OTLP token from the stack's OpenTelemetry tile. Treat as a secret; the value lives in Render, never the repo |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `http/protobuf` | the OTLP exporter's default, set explicitly so the blueprint carries the full contract |

### Web — Vercel

| Variable | Value | Notes |
|---|---|---|
| `NEXT_PUBLIC_API_URL` | `https://api.shipyard.yonatanem.com` | inlined at build — changing it requires a redeploy |
| `NEXT_PUBLIC_SENTRY_DSN` | shipyard-web project DSN | inlined at build; empty disables the reporter entirely |
| `SENTRY_ORG` · `SENTRY_PROJECT` | `yonatanemk` · `shipyard-web` | read by `withSentryConfig` |
| `SENTRY_AUTH_TOKEN` | organization token (Sentry → Developer Settings → Organization Tokens) | build-time only; uploads source maps. Treat as a secret |
| `NEXT_PUBLIC_POSTHOG_PROJECT_TOKEN` | the same PostHog project token | inlined at build; empty disables the reporter entirely |
| `NEXT_PUBLIC_POSTHOG_HOST` | `https://us.i.posthog.com` | ingestion origin, also inlined — a change needs a redeploy |

## 4. Deploy flow

1. Push/merge to `main`.
2. GitHub Actions (`ci.yml`) runs the quality gate: file policy → audit → lint → typecheck → format:check → build → tests.
3. Vercel builds `apps/web` (root directory `apps/web`; pnpm workspace install from the repo root) and deploys.
4. Render builds the API Dockerfile and deploys; the container start command runs `prisma migrate deploy` before the server starts.
- **Rollback:** Vercel — instant rollback to a previous deployment. Render free — rollback to either of the last two deploys.
- GHCR image publishing, SSH deploy and Caddy were retired with ADR-004.

## 5. Database & migrations

- Neon, one project, one database. `prisma migrate deploy` is idempotent and runs from the API container's start command (Render free has no pre-deploy step and no shell).
- Manual run (from a machine holding the production `DATABASE_URL`): `pnpm --filter @shipyard/api exec prisma migrate deploy`.
- Never hand-edit migrations; generate them in dev with `prisma migrate dev`.
- Free-plan facts: 0.5 GB storage, 100 CU-hours/month, scale-to-zero after 5 minutes (first query ~1 s), no IP allowlist (Scale-plan feature — compensating control: SSL + long password + credentials only in platform env).

## 6. Domains, DNS, TLS, OAuth

- Cloudflare DNS: `shipyard` → Vercel · `api` → Render · `assets` → R2 custom domain. Records are **DNS-only** (both platforms issue their own certificates; do not orange-cloud the app or API records).
- OAuth redirect URIs (Google allows a list; GitHub allows one URL per app — use a separate dev app):
  - Production: `https://api.shipyard.yonatanem.com/api/v1/auth/callback/google` and `…/callback/github`
  - Dev: `http://localhost:4000/api/v1/auth/callback/google` and `…/callback/github`
- R2 pre-production checklist (from ADR-004): attach `assets.yonatanem.com` to `shipyard-bucket`, set `R2_PUBLIC_BASE_URL` to it. No data migration — the DB stores object keys, not URLs.
- Email assets: templates reference the logo by absolute URL on the assets domain (a relative path only resolves in the react-email preview).

## 7. Health, observability, operations

- `GET /healthz` (liveness) · `GET /readyz` (readiness). Render's health check uses `/healthz`.
- Pino structured logs with request ids (Render log stream).
- **Sentry (as built 2026-09-23):** errors only — tracing, replays, logs and metrics stay off by design. The API reports what crosses the error middleware's *unexpected* branch (expected 4xx traffic stays in the logs), tagged `RENDER_GIT_COMMIT` as release and `NODE_ENV` as environment. The web app reports browser, server and edge errors, tagged with the Vercel commit SHA so the uploaded source maps resolve; its build uploads maps through `SENTRY_AUTH_TOKEN` and relays browser events through `/sentry-tunnel` so ad blockers do not eat them.
- **Privacy posture:** automatic collection is cut to what debugging needs — no request or response bodies, no bound SQL parameters, no user identity, no cookies. `Sentry.setUser()` is the one path by which user data could attach, and it is not called.
- **Product analytics (as built 2026-09-23):** PostHog (US region) carries the traffic and Core Web Vitals graphs and the nine product events declared in `packages/shared/src/analytics`. The browser reports pageviews, masked autocapture and Web Vitals; the API reports each event where its write commits, beside the existing business-event log line — so an event exists exactly when the thing it describes does, whichever door the request came through. Identity is the Shipyard user id, never an email or a name.
- **Analytics privacy posture:** ids and canonical enums only — no names, emails, issue titles or comment bodies; autocapture keeps the shape of an interaction but neither its text nor its attributes; a `before_send` hook rewrites one-time tokens out of pageview URLs, referrers and clicked hrefs (the invite / verify / reset links carry them in the path or the query); no token means no reporter, so tests and self-hosting stay silent. **Open decision:** EU visitors — add a consent banner, or run cookieless (`persistence: 'memory'`, losing person-level features; the org's US region cannot serve EU data residency either way).
- **Server telemetry (as built 2026-09-24):** OpenTelemetry in the API exports traces, metrics and logs over OTLP straight to Grafana Cloud — no collector in between, so the backend is an env var, not a rewrite. Traces: a span tree per request (HTTP → route → Prisma operation → pg query → outbound call) plus `email.send` and `mcp.tool_call` spans. Metrics: RED per route (`http_server_request_duration_seconds`), outbound calls, DB operation duration and pool (count, state, pending), runtime (event loop, heap), and one `email_sends_total{result}` counter. Logs: the pino lines themselves, carrying `trace_id`/`span_id` so Grafana jumps from a span to its logs and back. Every signal is tagged `service.name=shipyard-api`, `deployment.environment=NODE_ENV` and `service.version=RENDER_GIT_COMMIT`; the SDK is gated on `OTEL_EXPORTER_OTLP_ENDPOINT` being set, so tests and self-hosting stay silent. A `Shipyard API` dashboard covers RED, database, integrations and runtime.
- **Telemetry privacy posture:** span attributes and metric labels are ids, enums and route patterns only — never slugs, user or workspace ids, issue or comment content, or tokens; pino's redaction list keeps authorization headers and cookies out of the shipped lines too. Retention is the free tier's 14 days for all three signals.
- **Alerts:** an uptime monitor on `https://api.shipyard.yonatanem.com/readyz` every 5 minutes — it doubles as the keep-warm ping that keeps the free instance awake — plus issue alerts (new issue, regression; production environment only) delivered by email. Configured in the Sentry dashboard; nothing about them lives in the repo. Grafana Cloud adds three rule-based alerts, evaluated on the production environment only — 5xx share > 5% for 10m, p95 request duration > 1s for 10m, DB pool pending requests > 0 for 10m — delivered by email; configured in Grafana, nothing in the repo.
- **Cold starts:** Render free spins down after 15 idle minutes (~1 min wake). Optional keep-warm: a scheduled ping every ~14 minutes (fits inside 750 instance-hours/month; one always-awake service ≈ 744). The web middleware degrades to cookie presence when the API is unreachable, so page loads never hang on a cold API.
- Render may restart free services at any time — graceful shutdown (`SHUTDOWN_TIMEOUT_MS`) drains connections.
- Secrets live only in the platforms' environment settings; the repository ships `.env.example` only.

## 8. Backups, rollback, incidents

- **DB:** Neon managed backups/PITR (free-plan history window: 6 hours).
- **R2:** avatars/logos only — re-uploadable; no backup taken.
- **Rollback:** Vercel deployment rollback; Render deploy rollback (last two). Migrations are forward-only — roll back with a compensating migration, never by editing history.
- **Incident basics:** Render events + logs, Sentry issues, Grafana Cloud (traces, logs, dashboards), Neon dashboard; probe `/healthz` from outside to confirm liveness.

## 9. Self-hosting

The repo ships the dev `docker-compose.yml` (bundled Postgres) and the API Dockerfile. A self-hoster runs the API from the Dockerfile and the web app from source (`pnpm --filter @shipyard/web build && start`), pointed at their own Postgres/R2/Resend. The reference deployment above is the hosted variant; ADR-006's split means a self-hoster can also collapse everything onto one origin if they prefer, since the web app only needs `NEXT_PUBLIC_API_URL` to point somewhere reachable.

## 10. First-deploy smoke test

1. `GET /healthz` and `GET /readyz` from outside (expect 200).
2. Sign-up → verification email arrives → the link lands on `shipyard.yonatanem.com/verify-email?token=…` → signed in.
3. Sign-in (credentials) and one OAuth round-trip (Google or GitHub).
4. Invite flow → invitation email → accept → member visible.
5. Avatar upload renders from `https://assets.yonatanem.com`.
6. Password reset round-trip (email link → API validates → redirects to the web reset page).
7. An MCP client connects to `https://api.shipyard.yonatanem.com/mcp` with a workspace token and its writes appear in the activity log (F13 M9 acceptance).
8. Grafana Cloud shows the deployment's telemetry: traces, metrics and logs tagged `deployment_environment = "production"` (Explore → Tempo / Prometheus / Loki).
