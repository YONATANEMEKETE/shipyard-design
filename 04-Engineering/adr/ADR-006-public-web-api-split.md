# ADR-006: Public Web ↔ API Split — Direct Browser Calls

- **Status:** Accepted
- **Date:** 2026-09-22
- **Supersedes:** ADR-003 (Next.js proxy, API internal-only)
- **Amends:** ADR-005 (MCP server surface — its "one public door" premise)

## Context

ADR-003 kept the API internal-only and proxied every browser call through Next.js. That decision assumed what ADR-004 deployed: web and API as two containers in one Docker network behind Caddy. The hosting target changed to managed platforms with no shared network (ADR-007), so the same-network proxy no longer exists. Keeping the proxy by rewriting Next.js paths to an external API origin would route every API byte through the web platform's proxy layer and bandwidth accounting, re-introduce the buffering question for MCP streaming, and still leave the API hidden from the callers it now needs to serve — agents today, a mobile client later.

## Decision

**The API is a public, first-class origin. The browser calls it directly; nothing is proxied.**

```text
Browser ──▶ https://shipyard.yonatanem.com        (Vercel — Next.js)
        └─▶ https://api.shipyard.yonatanem.com    (Render — Express API, public)
Agent   ──▶ https://api.shipyard.yonatanem.com/mcp
```

- **Two named origins, one site.** Web `shipyard.yonatanem.com`, API `api.shipyard.yonatanem.com` — both under `yonatanem.com`, so browser calls are cross-*origin* but same-*site* and Better Auth's default `SameSite=Lax` cookies keep working.
- **Exact-origin CORS with credentials.** The API answers CORS only for `WEB_URL` plus an explicit `EXTRA_TRUSTED_ORIGINS` list (preview deployments, temporary hosts). No wildcard; requests without an `Origin` header (agents, curl, server-to-server) are unaffected. The same allowlist feeds Better Auth's `trustedOrigins` CSRF origin check — one list, one source of truth (the CORS middleware and the auth config read the same derived value).
- **The session cookie is shared across subdomains in production** (`COOKIE_DOMAIN=yonatanem.com` → Better Auth `crossSubDomainCookies`). The web origin's server-side session checks (Next middleware on protected routes) must see the cookie; a host-only cookie for `api.shipyard.…` would never reach `shipyard.…`.
- **The web app carries one API-origin value** — `NEXT_PUBLIC_API_URL`, inlined at build; every relative `/api/v1/…` path resolves against it in a single shared helper. The API carries `API_URL` as its own public base (Better Auth `baseURL`, OAuth redirect URIs) and `WEB_URL` for links and trust.
- **Redirects are absolute.** Better Auth resolves post-flow redirects (`callbackURL`, `redirectTo`) against its `baseURL` — the API origin — so the web app sends absolute web URLs (social sign-in, verify-email), and the API absolutizes the reset email's `callbackURL` against `WEB_URL` in its own email hook.
- **Email links keep their owners.** Verification links land on the web page (`{WEB_URL}/verify-email?token=…`, consumed client-side); the password-reset link stays on the API origin (its endpoint validates the token, then redirects to the web page).
- **MCP is reachable directly** at the API origin (amends ADR-005): agents were never browsers, and the proxy-buffering caveat disappears with the proxy.
- **The API is treated as a public surface.** Every route authenticates on its own (session cookie or bearer token); rate limits are per-IP and per-token; `/healthz` and `/readyz` are the only unauthenticated reads; the dev-only `/api/v1/test` router stays gated behind `NODE_ENV !== 'production'`.

## Alternatives considered

- **Keep the proxy via Vercel rewrites to the API origin** — zero web-side change and first-party cookies, but every API request and byte flows through the web platform (bandwidth accounting, edge/function limits), MCP streaming rides the proxy again, and the split's actual goal — a public API for agents and future clients — stays unmet. Rejected.
- **Keep the API internal-only behind a private path (VPC peering / tunnel)** — no free-tier path spans Vercel and Render; adds operational machinery for no user-visible benefit. Rejected.
- **One origin with a subpath (`yonatanem.com/api` proxied to the API)** — reintroduces a proxy and a third hop, and Vercel cannot do it without the same proxying this ADR removes. Rejected.
- **A separate apex domain for the API** — cookies become cross-*site* (`SameSite=None`) and third-party-cookie policies break the session in some browsers. Rejected; the shared registrable domain is the invariant.

## Reason

The proxy existed to buy one thing: a single public door on a self-hosted single box. Managed hosting (ADR-007) removes the box and with it the assumption. Keeping the proxy on Vercel would preserve the letter of ADR-003 while inheriting its costs and none of its benefits — and would leave MCP, a surface explicitly for non-browser callers, limping through a browser-oriented proxy.

The split makes the public surface honest: two origins, each doing one job, with the browser's cross-origin requirements (CORS, credentials, cookie scope, absolute redirects) satisfied deliberately and documented rather than implied.

## Consequences

- **Positive:** API traffic and any future streaming bypass the web platform entirely; agents and future mobile clients reach a stable public API; the web app deploys with a single API-origin value; CORS and the auth trusted-origin list read one derived allowlist; local dev keeps working with no special cases.
- **Negative / to manage:**
  - The API is publicly reachable — the attack surface ADR-003 halved is back by design. Compensating controls: authentication on every route, per-IP and per-token rate limits, exact-origin CORS (no wildcard), and `NODE_ENV`-gated test routes.
  - Cross-origin mechanics are load-bearing and easy to break: absolute callback URLs, the shared cookie domain, and the CORS allowlist must move together; integration tests pin each.
  - Preview deployments are random origins; each needs an explicit `EXTRA_TRUSTED_ORIGINS` entry and is UI-only until added.
  - The web middleware's session check degrades to cookie presence when the API is unreachable (unchanged behaviour) — with a cold API, this is what keeps page loads from stalling.
- **Revisit when:** a gateway/CDN in front of both origins becomes worthwhile (edge caching, WAF), or a second web client (mobile) needs its own origin in the allowlist.
