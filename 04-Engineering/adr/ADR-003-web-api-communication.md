# ADR-003: Web ↔ API Communication — Next.js Proxy

- **Status:** Accepted
- **Date:** 2026-08-12

## Context

The Next.js web app and the Express API run as two services in one Docker network (web :3000, api :4000). We must decide how the browser reaches the API, considering security, session cookies, future mobile reachability, and open-source self-hosting.

## Decision

**Next.js proxies all API calls; the API is internal-only.**

```
Browser ──HTTPS──▶ Caddy ──▶ Next.js (:3000) ──internal──▶ Express API (:4000)
```

- The API listens on the internal Docker network only; its port is **never published** to the host or the internet.
- Browser requests hit Next.js routes; server components / route handlers / a small fetch client in Next forward them to `http://api:4000/api/v1/...`, forwarding the session cookie.
- **No CORS** anywhere — there is no cross-origin browser traffic.
- In local dev, Next rewrites (`next.config` rewrites) proxy `/api/*` to `http://localhost:4000`, so the browser still only talks to Next.

## Alternatives considered

- **Direct browser → API calls (CORS-enabled public API)** — two public surfaces to secure; CORS config; auth tokens/headers handled client-side; rate limiting needed at both edges. Rejected for the MVP. The API can be deliberately exposed post-MVP when a public API becomes a product goal (PRD lists public APIs as out of scope).

## Reason

One public surface (Caddy → Next) is dramatically easier to secure: the session cookie stays HttpOnly and server-side, the API never needs CORS, and the attack surface halves. The API remains a clean internal service that a future mobile app can reach through a gateway without ever being loose on the internet. Cost: one extra in-network hop (~1ms on the same VPS — negligible).

## Consequences

- **Positive:** simpler security model, no CORS config, secrets stay server-side, cleaner story for the mobile-app future.
- **Negative:** the API is not directly browsable/demoable until a post-MVP decision to expose it; web and API must stay in the same network (true in compose and in local dev).
