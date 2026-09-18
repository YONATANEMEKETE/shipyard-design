# MCP Server — API Design

**Status:** Draft for review
**Last updated:** 2026-09-17
**Sources:** `features/mcp/spec.md` · `features/mcp/data-model.md` (locked — `mcp_token`, D1–D9) · `features/issues/api-design.md` (F5 precedent — list filters/sort, `SHIP-###` identifier lookup, archive semantics, error envelope) · `features/projects/api-design.md` · `features/cycles/api-design.md` · `features/comments/api-design.md` (F8 — authorship rules reuse) · `features/search/api-design.md` (F10 — grouped bounded reads) · `features/dashboard/api-design.md` (F9 — one composed read) · `features/activity/api-design.md` (emission is internal-only; pages are read-only) · `features/members/api-design.md` (F3 — role matrix, token-hash precedent) · `features/settings/api-design.md` (F11 — account-scoped surface, delegated sections as links) · `features/workspace/api-design.md` (F2 — `:slug` context, read-when-archived, leak-free 404s) · `features/auth/api-design.md` (F1 — session cookie on the browser surface only) · `00-architecture.md` §5–§8 · `ADR-001`–`ADR-003` · `ADR-005` (surface & hosting) · `05-Post-MVP.md`
**Owner:** `apps/api` — hand-written Shipyard code through the canonical pipeline (`route → validation → permission check → controller → service → repository → Prisma`), plus the MCP protocol layer (`features/mcp/`: transport validation → credential resolution → JSON-RPC dispatch → tool handler → owning service).

> **Protocol revision:** MCP `2026-07-28` (no `initialize` handshake, no sessions, no GET stream — every request is self-contained). Tool definitions are code, not data: the registry ships with the server binary.
>
> **Surface decision:** the MCP endpoint lives in `apps/api` and is reached publicly through the existing Next.js proxy (ADR-005) — one public door, no new deployable, tools call services in-process.

---

## 1. Base path & conventions

| Concern | Choice |
|---|---|
| MCP endpoint | `POST /mcp` — a **single** endpoint carrying JSON-RPC 2.0 messages. `GET` / `DELETE` ⇒ `405` (a modern-only server's answer to pre-`2026-07-28` clients). |
| Public path | `https://<web-host>/mcp`, rewritten by Next (`next.config.ts`) to `http://api:4000/mcp` next to the existing `/api/v1/:path*` rule. The API port stays unpublished. |
| Token management | `/api/v1/workspaces/:slug/agent-tokens` — workspace-scoped (binding a token requires a workspace + role context). Account settings links to it as a delegated section (F11 convention). |
| Transport headers | `MCP-Protocol-Version`, `Mcp-Method`, `Mcp-Name` (required, must match the body), `Accept` must list `application/json` **and** `text/event-stream`, `Authorization: Bearer <token>`. |
| Response mode | **`application/json` only in v1** (no SSE). The client must still be allowed to advertise both types; the server picks the shape per request, and every v1 tool completes synchronously. |
| Validation | Zod: `packages/shared/src/mcp/*` for tool argument contracts and token contracts; route-local schemas for the JSON-RPC envelope. Tool arguments are validated **before** any service call, and a validation failure is a *tool result*, not a protocol error (§8). |
| Auth transport | Bearer token only. Cookies are **never** accepted on `/mcp` (no browser session exists there — ADR-003 + ADR-005). |
| Workspace context | Resolved from the **credential**, not the URL: token → `(userId, workspaceId)` → live membership → the same `WorkspaceRequestContext` shape the cookie-guarded routes use (`workspaceId`, `memberId`, `slug`, `status`, `role`). |
| Statefulness | None. Every request re-resolves version, credential, membership, and scopes. No session id is ever minted or echoed; `Mcp-Session-Id` and `Last-Event-ID` are ignored if an old client sends them. |
| Envelope | Success: the JSON-RPC `result` of the requested method. Failure: either a JSON-RPC `error` object (protocol-level, §8.1) or a tool result with `isError: true` (domain-level, §8.2). |

---

## 2. Endpoint inventory

Four HTTP routes: one protocol endpoint plus three token-management routes (create/list, revoke, delete).

| # | Method | Path | Guard chain | Behavior |
|---|---|---|---|---|
| 1 | `POST` | `/mcp` | Origin check → header/body mirror validation → version check → bearer token → scope filter → JSON-RPC dispatch | `server/discover`, `tools/list`, `tools/call`. Unknown method ⇒ `404` + `-32601`. Notifications ⇒ `202` (the core protocol defines none client→server; tolerated and ignored). |
| 2 | `POST` | `/api/v1/workspaces/:slug/agent-tokens` | `requireSession` → `resolveWorkspaceContext(slug)` → any member | Create a token for **the caller**, bound to this workspace. Scopes requested must be within the caller's abilities (§3.2). Response contains the plaintext token **once**. |
| 3 | `GET` | `/api/v1/workspaces/:slug/agent-tokens` | `requireSession` → `resolveWorkspaceContext(slug)` → any member (own) / `OWNER\|ADMIN` (all) | List tokens: a member sees their own; Owner/Admin may pass `?all=true` to see every active token in the workspace. Never returns hashes or plaintext. |
| 4 | `POST` | `/api/v1/workspaces/:slug/agent-tokens/:tokenId/revoke` | `requireSession` → `resolveWorkspaceContext(slug)` → owner of the token, or `OWNER\|ADMIN` | Sets `revokedAt` (idempotent: revoking an already-revoked token returns `200`). |
| 5 | `DELETE` | `/api/v1/workspaces/:slug/agent-tokens/:tokenId` | `requireSession` → `resolveWorkspaceContext(slug)` → owner of the token, or `OWNER\|ADMIN` | Removes the row **and its hash** for good — the only way a revoked or expired connection leaves the list. Requires the product's standard `{ confirm: true }` body (missing ⇒ `400 VALIDATION_ERROR`). Not idempotent: a second call is `404 TOKEN_NOT_FOUND`, which the surface treats as done. |

> **Why one MCP route:** the protocol defines one endpoint and routes inside the JSON-RPC envelope. A route per tool would be a second, parallel API — exactly the duplication `packages/shared` exists to prevent.
>
> **Why token creation is any-member, not Owner/Admin:** a member's token can never exceed that member's own abilities (roles still gate every action at use time), so restricting issuance would only stop ordinary members from using agents while adding no security. Owner/Admin get the *revocation* power instead, because the real incident story is "a token leaked — kill it", not "members must not have tokens".
>
> **Why revocation is idempotent:** revocation is an emergency action; a double-click or a retried request must not fail.
>
> **Why deletion exists on top of revocation:** revocation is a timestamp — it stops the credential but keeps the row, because the row is the evidence ("was this used after I stopped using it?"). With no un-revoke in the product, that evidence accumulates forever, so deletion is the explicit housekeeping path: an `Active` token may be deleted too (the hash row leaves with it, so it rejects under the same one-predicate rule as a revoked one), which means "kill it and take it off my list" is one action instead of two.

### 2.1 Non-endpoints (explicitly not routes)

| Concern | Served by | Notes |
|---|---|---|
| A route per tool | — | Tools are JSON-RPC methods, not HTTP paths (§6) |
| Activity emission | Owning services (`activityService.record` in-transaction) | The MCP layer never writes activity directly; agents get no new emission path |
| Notifications | Owning services | Same recipients as the UI; no MCP-specific endpoints |
| Tool registry management | Code | No runtime registration, no DB rows (data-model §2.5) |
| Session endpoints | — | The protocol has no sessions; a credential is presented per request |
| Token editing | — | Tokens are immutable after creation except for revocation and deletion (scopes/label changes = revoke + create) |

---

## 3. Credential resolution & scope rules

### 3.1 The auth step (one query, one predicate)

```text
Authorization: Bearer shp_<...>
  → sha256(token)                     ← never the raw token
  → mcp_token row by hash             ← unique index lookup
  → rejected unless revokedAt IS NULL AND (expiresAt IS NULL OR expiresAt > now())
  → live workspace_member row for (userId, workspaceId)
  → WorkspaceRequestContext { workspaceId, memberId, slug, status, role }
```

Failure modes and their single answer:

| Case | Response |
|---|---|
| Header missing, malformed, or not a `shp_` token | `401` + `{ error: { code: "UNAUTHORIZED" } }` |
| Hash not found (never existed, deleted, or forged) | `401` — **identical** to the case above |
| Revoked | `401` — identical |
| Expired | `401` — identical |
| Token valid but membership no longer exists (member removed/left) | `401` |
| Token valid, membership live, but the action needs a scope the token lacks | Tool result, `isError: true`, naming the missing permission (§8.2) |

One message for every unusable-credential case: the holder of a stolen token learns nothing about *why* it stopped working, and there is exactly one code path to keep correct.

Two classes of answer, told apart by status:

- a problem with the **message** — malformed envelope, mirrored header disagreement, unsupported version, unknown method, unknown tool — is a JSON-RPC `error` (`-32020`, `-32022`, `-32600`, `-32601`, `-32602`, `-32700`);
- a problem with the **caller** — untrusted `Origin`, unusable credential, exhausted request budget — is the platform's own envelope: `403 FORBIDDEN`, `401 UNAUTHORIZED` (byte-identical to what the cookie path answers, so the two doors cannot be told apart), `429 RATE_LIMITED`.

Not in v1: the `WWW-Authenticate` header and the Protected Resource Metadata document. Both exist to point a client at an **authorization server**, and personal access tokens have none — a member pastes the token by hand. They arrive with OAuth (§12).

### 3.2 Scope issuance ceiling

A token's scopes are a subset of what its owner may do **at issuance time**:

| Requested scope | Allowed for |
|---|---|
| `READ` | Any member |
| `ISSUES_WRITE` | Any member (creating/editing issues is a member action — F5 rule 10) |
| `COMMENTS_WRITE` | Any member |
| `ISSUES_DELETE` | `OWNER` or `ADMIN` only (issue deletion is role-gated — F5 #7) |

Requesting a scope above the caller's role ⇒ `403` with a message naming the permission and the reason. This is a *convenience* ceiling, not the security boundary: the role check runs again on every action, so a member whose role is later downgraded loses the ability even though the token still carries the scope.

**`READ` is always granted.** A request that lists only a write scope comes back carrying `READ` as well — unioned at issuance, never rejected, because "let this connection edit issues" and "connections must be able to read" are separate statements and a `400` over a scope the member wanted anyway would be a worse surface (spec §3.1). The table above therefore describes what a member may **choose**, not the set a credential ends up with, and the ceiling still applies to the union: `READ` is added, nothing is elevated, and a member cannot reach `ISSUES_DELETE` by asking for it twice.

### 3.3 Management-route errors (implementation note)

The token-management routes answer with the standard HTTP envelope. Module-owned codes:

| Code | HTTP | When |
|---|---|---|
| `SCOPE_NOT_PERMITTED` | `403` | A creation request asked for a scope above the caller's role (§3.2). The message names the offending scopes and the caller's role |
| `TOKEN_EXPIRY_INVALID` | `400` | `expiresAt` is in the past. Rejected rather than minted dead-on-arrival, so the first `/mcp` request cannot fail with an unexplainable `401` |
| `TOKEN_NOT_FOUND` | `404` | The token id is unknown, belongs to another workspace, **or** belongs to another member while the caller is neither that member nor an Owner/Admin. All three answer identically so revocation (and deletion) cannot be used to probe other members' credentials |

Reused from the workspace module rather than duplicated: `WORKSPACE_NOT_FOUND` (unknown slug / non-member), `WORKSPACE_ARCHIVED` (creation in an archived workspace), `FORBIDDEN_ROLE`, `VALIDATION_ERROR`.

Per-route archived-workspace behaviour is deliberate: **create** requires an active workspace (`rejectArchived: true`), while **list**, **revoke** and **delete** also work while archived — a member must always be able to see, kill and clear out their credentials, and neither killing a leaked one nor clearing a dead one may be blocked by lifecycle state. `?all=true` is honoured for `OWNER|ADMIN`; any other member silently receives their own list, because the flag is a view request and never a permission claim.

---

## 4. Guard chain (canonical)

```text
POST /mcp
  │
  ├─ Origin validation            ← reject untrusted Origin with 403 (mandatory for HTTP transports)
  ├─ Envelope validation          ← JSON-RPC shape; _meta.protocolVersion present
  ├─ Header/body mirror check     ← MCP-Protocol-Version, Mcp-Method, Mcp-Name must equal the body ⇒ 400 + -32020
  ├─ Version support              ← unsupported ⇒ 400 + -32022 with the supported list
  │
  ├─ Credential resolution        ← bearer → hash → token row → live membership → WorkspaceRequestContext (§3.1)
  │                                (reject ⇒ 401; no context ⇒ no dispatch)
  │
  ├─ Scope filter                 ← tools/list returns only what the scopes allow; tools/call for a
  │                                scope-less tool ⇒ readable tool result (§8.2)
  │
  └─ JSON-RPC dispatch
        server/discover → static description of this server
        tools/list      → registry ∩ scopes, deterministic order, ttlMs
        tools/call      → Zod argument validation → tool handler
                            → owning module's SERVICE (never a repository, never HTTP to self)
                            → map result | map AppError → tool result
```

The two doors that matter on every call: **the credential** (who) and the **live membership** (still allowed?). Neither is cached in a way that outlives a request.

---

## 5. Protocol contract

### 5.1 Required headers (all requests)

| Header | Rule | Failure |
|---|---|---|
| `Content-Type: application/json` | The body is one JSON-RPC request or notification | `400` |
| `Accept` | Must list `application/json` and `text/event-stream`. **Lenient server policy:** v1 answers JSON regardless, and does not reject a client that omits `text/event-stream` | — |
| `MCP-Protocol-Version` | Must equal `_meta["io.modelcontextprotocol/protocolVersion"]` | `400` + `-32020` |
| `Mcp-Method` | Must equal the body's `method` | `400` + `-32020` |
| `Mcp-Name` | For `tools/call` only: must equal `params.name` | `400` + `-32020` |
| `Authorization` | Bearer token | `401` |

Mirror validation is mandatory (the confused-deputy rule): any disagreement between a mirrored header and the body is rejected, and a value that arrives base64-wrapped in the `=?base64?…?=` sentinel is decoded before comparison. v1 uses **no** `x-mcp-header` parameters, so no `Mcp-Param-*` headers exist to validate — the mechanism is documented here for the day a proxy needs one.

**Era note (M6):** `MCP-Protocol-Version` is required in **both** eras, but the three mirror checks above belong to `2026-07-28` and therefore apply to modern requests only — a legacy client has no `Mcp-Method` or `Mcp-Name` to send, so holding it to them would be a contract its revision never had (§5.6).

### 5.2 Status codes

| Situation | Response |
|---|---|
| Request handled (success or tool-level failure) | `200` + `application/json` |
| Notification received | `202`, empty body |
| Header/body mismatch, missing required header, malformed envelope | `400` (`-32020` / `-32600`) |
| Unsupported protocol version | `400` + `-32022` with `supported` |
| Missing/invalid/revoked/expired credential, or membership gone | `401` |
| Untrusted `Origin` | `403` |
| Unknown JSON-RPC method | `404` + `-32601` |
| `GET` / `DELETE` | `405` |
| Edge rate limit | `429` (transport-level only; per-token limits are tool results — §8.2) |

### 5.3 `server/discover`

Returns `supportedVersions: ["2026-07-28"]`, `capabilities: { tools: {} }`, `serverInfo` (name/version), an `instructions` string (the one place cross-tool guidance lives — "prefer the identifier tool when you have a `SHIP-###`; use search when you don't; never invent identifiers"), and cache metadata. No `resources`, no `prompts`, no extensions in v1 — the surface declares exactly what exists.

### 5.4 `tools/list`

- Returns the registry **filtered by the token's scopes** (a `READ` token sees no write tools).
- **Deterministic order** (fixed registry order) — stable prefixes improve client-side caching and prompt-cache hits.
- Carries `ttlMs` (long — the registry is static) and `cacheScope: "private"` (the list depends on the credential).
- Tool names are prefixed `shipyard_` and unique within this server.
- Malformed tool definitions are logged and skipped rather than failing the whole list.

### 5.5 `tools/call`

- `params.name` must exist in the registry **and** be permitted by the caller's scopes; otherwise §8.
- Arguments are validated against the tool's Zod contract (shared with the HTTP contracts where they overlap — one shape, no parallel definitions).
- The handler builds the `WorkspaceRequestContext` (already resolved) → calls the owning service → maps the outcome.
- Results are `resultType: "complete"` with `content` (text) and, where the shape is stable, `structuredContent`. Errors are `isError: true` with an actionable sentence (§8.2).
- **No elicitation / `input_result` in v1**, so no multi-round-trip resumability is required. When phase 3 adds human confirmation, the MRTR flow arrives with it (api-design §12).

### 5.6 The two eras

The endpoint serves **two revisions on one URL** (ADR-005 amendment):
`2026-07-28` (preferred) and `2025-11-25` — the newest revision of the era before
it, and the newest that any shipped client negotiates. Every request is classified
from the request itself, because the server is stateless and has no session to
remember an era in.

| Signal in the request | Era it means | Why |
|---|---|---|
| `method` is `initialize` or `notifications/initialized` | legacy | Those methods do not exist in `2026-07-28`, so sending one *is* the declaration — whatever revision the client proposes |
| `params._meta` present | modern | Outranks the header, so a request cannot escape the mirrored-header checks by putting a legacy revision in a header. `_meta` naming a legacy revision is served as modern: stricter rules, nothing skipped |
| `MCP-Protocol-Version` header only | that revision's era | The post-handshake shape of the legacy era |
| Nothing at all | — | `400` + `-32600` naming `params._meta`. Deliberately **not** "assume the oldest era" — a modern client that forgets its envelope must be told |
| A revision we do not speak | — | `400` + `-32022` with `supported` + `requested`, answered **before** the envelope rules: a legacy request has no `_meta` to fail, so the version answer is the only one it can act on |

Per era:

| | modern `2026-07-28` | legacy `2025-11-25` |
|---|---|---|
| Opens with | `server/discover` | `initialize` → `notifications/initialized` (answered `202`) |
| Revision & identity | per request, in `params._meta` | once, in the handshake — `serverInfo` top-level |
| Mirrored headers | required, and must agree with the body | **not applicable**: those fields do not exist in that revision |
| Session | none | none — the handshake answer carries no `Mcp-Session-Id` |
| Results | `resultType`, `ttlMs`, `cacheScope` | the same content with that frame projected away; `_meta` diagnostics kept |
| `GET` / `DELETE` | `405` | `405` (that revision permits it; the measured client accepts it) |
| `server/discover` | served | not served — that era never knew the method |

Both eras carry the **same** credential rules, the same `Origin` guard, the same
per-token budget, and the same error envelopes — a refusal is identical whichever
door was used. The era changes the *frame*, never the permission model.

---

## 6. Tool inventory

Names, scopes, annotations and backing services. `RO` = `readOnlyHint`, `DES` = `destructiveHint`, `IDEM` = `idempotentHint`, `OW` = `openWorldHint` (all `false` for this server — nothing here reaches outside Shipyard).

### 6.1 Phase 1 — reads (scope `READ`)

| Tool | Purpose | Arguments | Hints | Owning service |
|---|---|---|---|---|
| `shipyard_list_issues` | Browse/filter active work | `status[]`, `priority[]`, `assignee` (name/email/id or `me`), `project`, `cycle`, `labels[]`, `blocked`, `dueBefore`, `dueAfter`, `includeArchived`, `sort`, `order`, `limit` (default 25, max 50) | RO | `issuesService.list` |
| `shipyard_get_issue` | One issue in full + recent history | `issue` (`SHIP-###` or id), `includeHistory` | RO | `issuesService.getDetail`, `listHistory` |
| `shipyard_search` | Find things when the identifier is unknown | `query` (required, 1–200), `type` (issues/projects/cycles/members/comments), `limit` (default 10, max 50) | RO | `searchService` |
| `shipyard_list_projects` | Project overview | `status`, `includeArchived`, `limit` | RO | `projectsService.list` |
| `shipyard_list_cycles` | Cycle overview (which is active, dates, goals) | `status`, `includeArchived`, `limit` | RO | `cyclesService.list` |
| `shipyard_workspace_overview` | Orientation in one call: my work, active projects, current cycle, blocked/overdue, recent activity | *(none)* | RO | `dashboardService` |
| `shipyard_recent_activity` | Who did what lately | `area`, `actor`, `limit`, `cursor` | RO | `activityService` |
| `shipyard_list_members` | Resolve assignees and roles | `limit` | RO | `membersService` |

### 6.2 Phase 2 — additive writes (scope `ISSUES_WRITE` / `COMMENTS_WRITE`)

| Tool | Purpose | Arguments | Hints | Owning service |
|---|---|---|---|---|
| `shipyard_create_issue` | File new work | `title` (required), `description`, `priority`, `status`, `assignee`, `project`, `labels[]`, `dueDate` | RO `false`, DES `false`, IDEM `false` | `issuesService.create` |
| `shipyard_update_issue` | Change content/planning | `issue` (required), `title`, `description`, `priority`, `dueDate`, `project` (`null` detaches), `cycle` (`null` detaches) | RO `false`, DES `false`, IDEM `true` | `issuesService.update` |
| `shipyard_set_issue_status` | Move between statuses | `issue`, `status` (`BACKLOG`/`TODO`/`IN_PROGRESS`/`DONE`) | RO `false`, DES `false`, IDEM `true` | `issuesService.update` |
| `shipyard_assign_issue` | Assign or unassign | `issue`, `assignee` (name/email/id, or `null` to unassign) | RO `false`, DES `false`, IDEM `true` | `issuesService.update` |
| `shipyard_block_issue` | Set/clear blocked state | `issue`, `blocked` (bool), `reason` | RO `false`, DES `false`, IDEM `true` | `issuesService.update` |
| `shipyard_add_comment` | Comment on an issue | `issue`, `body` | RO `false`, DES `false`, IDEM `false` | `commentsService.create` |

### 6.3 Phase 3 — gated lifecycle (scope `ISSUES_DELETE` + `OWNER|ADMIN`)

| Tool | Purpose | Arguments | Hints | Owning service |
|---|---|---|---|---|
| `shipyard_archive_issue` | Reversible removal from active work | `issue` | RO `false`, DES `false`, IDEM `true` | `issuesService.archive` |
| `shipyard_restore_issue` | Return an archived issue to its previous state | `issue` | RO `false`, DES `false`, IDEM `true` | `issuesService.restore` |
| `shipyard_delete_issue` | Permanent deletion | `issue` (+ human confirmation per spec §3.4) | RO `false`, DES `true`, IDEM `true` | `issuesService.remove` |

### 6.4 Implementation notes (M5)

Shipped with the eight read tools. These are the decisions the tables above do
not carry.

- **One contract, two consumers.** Each tool's arguments are a Zod schema in
  `packages/shared/src/mcp/tools.ts`. The handler validates with that schema, and
  the JSON Schema advertised in `tools/list` is **generated** from it
  (`z.toJSONSchema`), with properties carrying defaults removed from `required` —
  a model should not be told it must send every default when omitting it is the
  intended path.
- **`includeArchived` means "as well as".** The list queries answer one state per
  call, so a tool asking for both makes two calls and concatenates: active first,
  archived after. `total` is the sum, so the truncation count stays honest.
- **Names are input; ids are never required.** `assignee` takes a name, an email,
  or `me`; `project`, `cycle` and `labels` take names (labels also by id).
  Resolution happens in the tool layer with one members/projects/cycles/labels
  read per call — the services keep taking ids, as the HTTP routes do.
- **An identifier is what a person says.** `SHIP-42` resolves through the same
  `q` filter the UI's search box uses (the list service maps it to `seqNumber`);
  an internal id is accepted too. A one-character identifier resolves to nothing
  rather than to an arbitrary issue, because that filter ignores queries shorter
  than two characters. Reading by identifier is not a browse: an archived issue
  is returned, and the result says it is archived.
- **Emails are matched, never returned.** An address may be used as *input*
  because that is how people refer to each other; `shipyard_list_members` answers
  with name + role only.
- **The scope gate is at dispatch.** `tools/call` re-checks the credential's
  scopes against the registry entry before the handler runs, so a tool that was
  not advertised is not callable — the scope list is a gate, not advice.
- **Each call logs keys, not values** (§10): tool name, argument *names*,
  duration, result size, `isError`, token id.

**Deliberately absent:** member invitations/roles/ownership, workspace lifecycle, account settings, notification management, label/project/cycle deletion, bulk operations, and any generic "make an HTTP request" capability. Absence is a design decision (spec §6), not a backlog.

---

## 7. Result shaping rules

| Rule | Detail |
|---|---|
| Compact lists | List tools return card projections (identifier, title, status, priority, assignee name, project/cycle name, due date, labels) — never descriptions, comment bodies, or internal ids beyond what actions need |
| Bounded | Every list tool takes `limit` with a server-side maximum; the server always applies min(requested, max) |
| Honest truncation | List results include `returned`, `truncated`, and `total` where the source knows its own size; the text says which of the two situations it is: with a next page available, "25 shown — pass the cursor for the next page"; otherwise "25 of 84 shown — narrow the filters or raise the limit". Only `shipyard_recent_activity` pages by cursor in v1 |
| Identifiers | `SHIP-###` is returned verbatim and accepted as input; internal ids are accepted but never required |
| Minimal PII | Member projections return name + role (+ id for actions). **Email is not exposed** on the agent surface (spec §7 Q2) |
| Deterministic | Same filters ⇒ same order; ties broken by id |
| Text + structure | Every result carries a short human-readable text block; machine-stable shapes also carry `structuredContent` |
| No secrets | Tool results never include tokens, hashes, `tokenPrefix`, or another member's private fields |

---

## 8. Error mapping

### 8.1 Protocol errors (JSON-RPC `error`)

Reserved for calls that cannot be understood at all.

| Code | When |
|---|---|
| `-32020` | Mirrored header ≠ body, or a required header is missing/malformed |
| `-32022` | Unsupported `MCP-Protocol-Version` (payload lists `supported`) |
| `-32600` | Malformed JSON-RPC envelope or missing required `_meta` |
| `-32601` | Unknown method (`404`) |
| `-32602` | `tools/call` naming a tool that does not exist in the registry |

### 8.2 Tool results (`isError: true`) — every domain failure

| Domain outcome | HTTP-envelope analogue | Tool result text must say |
|---|---|---|
| Zod validation failure on arguments | **Tool result**, `isError: true`, `INVALID_ARGUMENTS` in `_meta` | Every failing argument, not just the first: its name, the bound or enum it broke, and an invitation to call again with corrected arguments. Deliberately *not* a JSON-RPC error — the message was understood, and the model is the party that can fix it, so a 4xx would only teach it to abandon a tool it is allowed to use |
| Missing scope | `403` (conceptually) | Which permission the token lacks and that a new token or a UI action is needed |
| Role failure (`FORBIDDEN_ROLE`) | `403` | The required role, the caller's role, and what they can do instead |
| Not found / cross-workspace (`ISSUE_NOT_FOUND`, `*_NOT_IN_WORKSPACE`) | `404` | The identifier that failed + "verify it in this workspace". **Identical** wording for a non-existent and a non-visible resource |
| Archived resource (`ISSUE_ARCHIVED`, `*_ARCHIVED`) | `409` | The resource is read-only and the restore action to call first |
| State conflicts (`CYCLE_OVERLAP`, `ANOTHER_ACTIVE_EXISTS`, `LABEL_NAME_CONFLICT`, `TRANSFER_REQUIRED`, …) | `409` | The blocking state + the exact next call |
| Workspace archived | `409 WORKSPACE_ARCHIVED` | Reads still work; writes need the workspace restored |
| Rate limited (per token) | `429` (conceptually) | Retry-after hint, so the agent paces instead of hammering |
| Unexpected failure | `500` | A generic sentence + the request id. Never a stack trace, SQL, or a Prisma message |

Rules: a domain failure is **never** a protocol error; a protocol error is **never** a tool result; no failure message discloses what the caller may not see; and every message is written for the consumer that will read it — the model.

---

## 9. State matrix

| Situation | Reads | Writes | Destructive |
|---|---|---|---|
| Token active, workspace active | ✅ | ✅ (scopes permitting) | ✅ if scoped + role |
| Token active, workspace `ARCHIVED` | ✅ (mirrors read-when-archived) | ❌ `409 WORKSPACE_ARCHIVED` as a tool result | ❌ |
| Token revoked / expired | ❌ `401` | ❌ `401` | ❌ `401` |
| Member removed from the workspace | ❌ `401` | ❌ `401` | ❌ `401` |
| Issue archived | ✅ read | ❌ `ISSUE_ARCHIVED` + restore hint | ✅ `OWNER\|ADMIN` (delete allowed regardless of archive state — F5 #7) |
| Read-only token | ✅ | ❌ tool not even listed; direct call ⇒ readable scope failure | ❌ |
| Member role downgraded, token scopes unchanged | ✅ | Depends on the new role — the role check runs live | ❌ |

---

## 10. Rate limiting, logging & redaction

| Concern | Choice |
|---|---|
| Edge limit | Per-IP limit at Caddy/Express (protects the box) |
| Per-token limit | App-level, counted per **token id** — the credential is the only identity on this surface, and agents share an IP with everything else on the machine, so per-IP is not fairness. Fixed window, `MCP_RATE_LIMIT_WINDOW_MS` / `MCP_RATE_LIMIT_MAX` (default 60s / 120). Per-process store, like the API's own limiter; a shared store is a deployment concern (M9) |
| Limit breach | `tools/call` ⇒ a tool result with a retry hint (§8.2), so the model paces itself; every other method ⇒ `429 RATE_LIMITED` with `Retry-After` and `details.retryAfterMs` — discovery has no tool-result channel to carry the hint, and a bare `429` with no hint is a dead end. Nothing is counted before the credential resolves, so a refused request cannot exhaust anyone's budget |
| Log fields | request id, token id (not hash, not plaintext), user id, workspace id, tool name, argument **keys** (not values), duration ms, result bytes, `isError` |
| Never logged | The raw token, the hash, `Authorization` header contents, member emails, full tool argument values |
| `lastUsedAt` | Best-effort, at most one write per minute, outside the tool's transaction (data-model D6) — the throttle is a condition in the `WHERE` of a single statement, so concurrent calls cannot both stamp it and a failed stamp never fails the call |
| Redaction | The `shp_` prefix makes the pattern greppable in logs, scanners, and error reports |

---

## 11. Tests

| Layer | Assertions |
|---|---|
| **Unit** (`test/unit/mcp/`) | Error mapper (`AppError` → `isError` + code in `_meta`; unknown → generic + request id, no stack); bearer parsing (scheme case-insensitive, another scheme / whitespace inside the value / bare scheme ⇒ not a credential); the per-token budget (allow up to the max, refuse after, reset when the window passes, one budget per token, nothing counted before resolution); portmanteau argument mapping (`me`, `SHIP-###` → `(workspaceId, seqNumber)`, cuid passthrough); scope filter removes write tools by name; scope issuance ceiling per role |
| **Transport** (integration) | Untrusted `Origin` ⇒ `403` + `FORBIDDEN`; `GET`/`DELETE` ⇒ `405`; missing `MCP-Protocol-Version` ⇒ `400` + `-32020`; `Mcp-Name` ≠ body ⇒ `400` + `-32020`; unknown version ⇒ `400` + `-32022`; unknown method ⇒ `404` + `-32601`; notification ⇒ `202`; no credential, and a revoked one ⇒ `401` + `UNAUTHORIZED` **before dispatch**; the (max+1)th call for one token ⇒ `429` + `Retry-After`, while a second token from the same client is unaffected |
| **Credentials** (integration, Testcontainers Postgres) | Every unusable credential — missing/empty header, wrong scheme, not a token, unknown hash, revoked, expired, deleted, membership removed — asserted **identical** (status, code, message); a session cookie alone authenticates nothing; a valid token's context **equals** the one the cookie path builds for the same member; `lastUsedAt` stamped once per window and never on a refused request; no log record contains the token, its prefix, or its hash |
| **Protocol & tools** (integration, Testcontainers Postgres) | `server/discover` shape; `tools/list` deterministic order + `ttlMs` + `cacheScope: private` + scope pruning by name; every tool's happy path asserted **against the database**; writes assert history/activity/notification rows; cross-workspace identifier ⇒ same flat not-found as a fake one; archived issue ⇒ actionable tool result; role failure ⇒ actionable tool result; per-token limit ⇒ retry hint |
| **Legacy era** (integration) | `initialize` answered statelessly — our revision, `serverInfo` top-level, **no** `Mcp-Session-Id` in header or body — against the request shape measured from the shipped client; `notifications/initialized` ⇒ `202`; `tools/list` and `tools/call` in that era's envelope (no `resultType`/`ttlMs`/`cacheScope`) with `_meta` diagnostics preserved; an unspoken revision in the header ⇒ `400` + `-32022` with the supported list; a revoked credential and a removed membership ⇒ the **same** `401` as the modern door, compared field by field; `GET`/`DELETE` ⇒ `405`; and the modern path still enforces both mirrors |
| **Not testable** | Whether an agent *chooses* the right tool or whether a result is the right size — measured by dogfooding with a real client over the endpoint (the legacy era exists so a shipped client can connect) plus the call log (§10), then fixed in the tool definitions. Not a week-long programme: one session of real questions, then the definitions that failed it |

---

## 12. Deferred (with triggers)

| Item | Trigger |
|---|---|
| SSE responses (`text/event-stream`) | A tool that benefits from progress (e.g. bulk import) — today everything completes synchronously |
| Human confirmation via elicitation / MRTR | Phase 3 destructive tools (spec §3.4): host-driven consent first, in-protocol confirmation second |
| `subscriptions/listen` for tool-list changes | Only if the registry ever becomes dynamic per workspace/role |
| OAuth 2.1 (resource-server role, PRM, client metadata docs, audience validation) | External agent clients that cannot paste a token (spec §7 Q1) |
| Per-tool permissions beyond the four scopes | Evidence that scopes are too coarse |
| `x-mcp-header` parameters | A gateway/proxy that must route or throttle on an argument — and then the workspace-selector case is explicitly *not* it (the workspace is bound to the token) |
| Tool-call audit table | A member-visible agent-activity view or retention need (data-model D9) |
| MCP-UI / task / skill extensions (`io.modelcontextprotocol/ui`, `…/tasks`, `…/skills`) | The shipped client advertises all three in its handshake (measured, M6 probe). We ignore them and return text blocks, which is the v1 answer; revisit when a member-visible UI inside the agent, long-running task semantics, or packaged skills are actually asked for |
