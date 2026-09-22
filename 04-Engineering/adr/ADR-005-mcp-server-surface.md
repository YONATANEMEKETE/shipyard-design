# ADR-005: MCP Server Surface & Hosting

- **Status:** Accepted — amended by ADR-006 (2026-09-22)
- **Date:** 2026-09-17
- **Related:** `ADR-003` (superseded by `ADR-006` — public web/API split) · `05-Post-MVP.md` (MCP server as the first post-MVP milestone)

> **Amendment (2026-09-22, ADR-006):** the MCP endpoint is reachable **directly** at the API origin — `https://api.shipyard.yonatanem.com/mcp`; the Next.js rewrite is gone with the rest of the proxy. Everything else in this ADR stands: the module lives in `apps/api`, credentials are workspace-bound personal access tokens, responses are JSON, and tools call services in-process. The "one public door" rationale below was superseded by ADR-006's deployment reasoning, and the alternative it rejected — publishing the API directly — is now, in effect, the chosen topology for the whole API.

## Context

Shipyard will expose an **MCP server** so a member's AI agent can work in a workspace on that member's behalf (`features/mcp/spec.md`). The protocol's remote transport is HTTP, so the endpoint must be reachable by programs that are **not** the browser:

- no cookie jar, no page on `shipyard.yonatanem.com`, no session;
- credentials presented per request (`Authorization: Bearer …`);
- possibly many callers per IP.

ADR-003 deliberately made the API **internal-only** behind a single public door (Caddy → Next.js → internal Express API), on the reasoning that one public surface halves the attack surface. That decision assumed the only external caller was our own web app in a browser. MCP breaks that assumption, so something must change — either a second public surface appears, or the existing one is extended to carry this traffic.

We must also decide where the MCP code lives, because the tools need the domain rules (validation, transactions, permissions, notifications) that live in `apps/api`'s services.

## Decision

**The MCP endpoint is implemented inside `apps/api` and exposed through the existing Next.js proxy. Credentials are workspace-bound personal access tokens. Responses are JSON-only in v1.**

```
Agent ──HTTPS──▶ Caddy ──▶ Next.js (:3000) ──rewrite──▶ Express API (:4000, internal)
                                │                              │
                      /api/v1/*  →  API                   features/*   (existing services)
                      /mcp       →  API  /mcp              features/mcp (new: transport → rpc → tools)
```

- **One public door preserved.** Caddy still exposes only `web:3000`; `/mcp` is forwarded by a rewrite next to the existing `/api/v1/:path*` rule. The API port stays unpublished.
- **No new deployable.** The MCP layer is a feature module in the API (`routes → rpc → tool handler → owning service`), so tools call services **in-process** — no HTTP hop to self, no duplicated business rules.
- **Auth is bearer-only** and bound to exactly one workspace: the workspace is a property of the credential, never a request argument (`features/mcp/api-design.md` §3).
- **JSON responses in v1.** No SSE, so no long-lived streams through the proxy; the server picks the response shape per request, so streaming can be added later without changing the endpoint.
- **PATs first, OAuth later.** Authorization is optional in MCP; tokens pasted into an agent's config are sufficient for the MVP, and the OAuth resource-server flow is a documented later milestone.

## Alternatives considered

- **A Next.js route handler for `/mcp`** — keeps "only Next is public" literally true, but the domain logic lives in `apps/api`, so the handler would have to call the API over HTTP anyway; the API would still need an MCP-shaped surface internally, and we'd have added a hop and a second place where the protocol layer half-lives. Rejected as strictly more work for the same topology.
- **A separate `apps/mcp` service** — cleaner isolation, its own scaling, release cadence and rate limits; but it cannot import `apps/api`'s services, so it must either call the API over HTTP or wait for the domain logic to be extracted into a shared package (`packages/core`). That extraction is real work with real risk (every service, every repository, every transaction boundary), and the MVP gains nothing from it. Rejected **now**; retained as the documented escape hatch if the surface ever needs independent scaling or heavy streaming.
- **Publishing the API directly and serving `/mcp` from it** — the API becomes a second public surface, with its own TLS, CORS decisions, edge hardening and rate-limit story, and the browser-facing cookie model sits on the same process as token traffic. Rejected: it discards ADR-003's core rationale for no benefit.

## Reason

ADR-003's decision was about **how many doors**, not which process owns the code. Extending the existing door keeps every benefit ADR-003 bought (no CORS, secrets server-side, one TLS terminator, one place where untrusted traffic enters) while satisfying MCP's actual requirement — that a non-browser program can reach the endpoint.

Putting the code in the process that owns the domain rules is what makes the tool layer thin: a tool parses arguments, checks its credential and scope, calls an existing service, and maps the result. Business rules stay where they already live, and the "no agent shortcut" spec rule is satisfied structurally rather than by discipline.

## Consequences

- **Positive:** one public surface preserved; no new service to build, deploy, monitor, or pay for; tools reuse services, transactions, logging, request ids and Prisma; bearer tokens are local to the new module; the whole feature is testable with the existing Supertest + Testcontainers harness.
- **Negative / to manage:**
  - The API now processes **untrusted public traffic** through the proxy, so `Origin` validation, header/body mirror validation, per-token rate limits and log redaction are required work, not optional hardening (`features/mcp/api-design.md` §5, §10).
  - Next must forward machine-caller headers faithfully (`Authorization`, `MCP-Protocol-Version`, `Mcp-*`). This gets an explicit end-to-end test, not an assumption.
  - Streaming, if it ever lands, may be buffered by the proxy — the mitigation is a direct Caddy route for `/mcp` or a streaming-capable forwarding path, decided when a tool actually needs SSE.
  - `/mcp` needs its **own** rate-limit policy rather than inheriting the `/api/v1` one, because agents share IPs.
  - The MVP ships no sessions and no GET stream (current protocol revision), so old clients hitting `GET`/`DELETE` get `405` and any `Mcp-Session-Id` is ignored (`features/mcp/api-design.md` §5).
- **Revisit when:** the MCP surface needs independent scaling or release cadence, requires long-lived streams as a core feature, or external clients need OAuth-based connection — at which point move the module to `apps/mcp` with a `packages/core` extraction (`features/mcp/api-design.md` §12).

## Amendment (M6): the legacy era is served too

**Changes the Decision above in one place:** the surface no longer speaks a single
revision.

What forced it, measured rather than assumed: a throwaway server answering a
legacy handshake and logging every request, driven by
`@modelcontextprotocol/inspector@2.7.0` (which bundles
`@modelcontextprotocol/sdk` 1.30.0 — newest revision `2025-11-25`). **No shipped
client negotiates `2026-07-28`.** A modern-only server therefore cannot be
connected to by a real agent at all, which makes M6's own gate (dogfooding) and
M9's external-client story impossible without either a second era or a bridge we
would have to build, ship and maintain.

The decision: **both eras, one endpoint, stateless in both.**

- `initialize` is answered with a handshake carrying **our** legacy revision
  (`2025-11-25`) and `serverInfo` top-level, with **no `Mcp-Session-Id`** — that
  revision permits a session-less server, the shipped client was measured
  accepting one, so the original "no sessions" decision survives intact.
- The era is derived per request from the request itself
  (`mcp/transport.ts: detectRequestEra`): the handshake names its era outright,
  `params._meta` means modern and outranks the header, otherwise the version
  header decides, and no version signal at all is an error rather than an
  assumption.
- **Accepted cost.** The mirrored-header rule (`MCP-Protocol-Version`,
  `Mcp-Method`, `Mcp-Name` agreeing with the body) is a `2026-07-28` control, and
  a legacy request has no such fields to send. Era support therefore weakens the
  confused-deputy posture *for requests that declare the older era*, and for those
  only. On the modern path every check is unchanged, and both halves are asserted
  in the transport suite. Everything era-independent — the `Origin` guard,
  credential resolution, the per-token budget, the error envelopes, and the
  rule that a refusal is identical whichever door was used — applies to both.
- **Dropped:** refusing `initialize` with a version diagnostic. That message
  existed to explain a door that did not open.

**Revisit when:** every client we care about speaks `2026-07-28`. Then delete the
legacy branch, its schemas and its tests, and the mirror rule becomes
unconditional again.
