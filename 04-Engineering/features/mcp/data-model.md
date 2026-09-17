# MCP Server — Data Model

**Status:** Draft for review
**Last updated:** 2026-09-17
**Sources:** `features/mcp/spec.md` · `features/settings/data-model.md` (F11 precedent — one new table, zero edits to generated tables; D1 isolation principle reused here) · `features/auth/data-model.md` (F1 — `user` table is Better Auth-owned; `user.id` is the opaque identity used by FKs, never hand-modified) · `features/members/data-model.md` (F3 precedent — cascade convention, service-owned invariants, partial/functional indexes appended in raw SQL when Prisma cannot express them) · `features/workspace/data-model.md` (F2 — `workspace.id` ownership, archived-workspace semantics) · `features/activity/data-model.md` (actor name is frozen into the event at write time; a provenance column is considered there, not here — D8) · `features/notifications/data-model.md` (F6 — recipient-scoped reads, no new columns needed for agent actions) · `00-architecture.md` §5/§8/§9 (module boundaries, data-layer strategy, 12-factor config) · `ADR-001` (Prisma + Postgres) · `ADR-002` (shared contracts) · `ADR-005` (MCP surface) · `05-Post-MVP.md`
**Owner:** `apps/api` — Prisma-owned (hand-modeled, like workspace/members/projects/issues/cycles/comments/activity/settings).

> **Locked scope (2026-09-17):** **one** new table (`mcp_token`) and **one** new enum (`McpTokenScope`). No edits to any existing table, including no provenance column on the activity log (D8) and no audit table for tool calls (D9). Tokens are workspace-bound and member-owned; stored as a hash only; revoked by timestamp, never deleted; `lastUsedAt` maintained best-effort (D6). The MCP endpoint itself stores **nothing** — it is stateless and derives context per request from the credential.

---

## 1. Overview

MCP Server owns exactly one kind of durable state: **credentials**. Everything else about the feature — the tool registry, the JSON-RPC layer, per-request workspace resolution — is computed at request time from existing tables (membership, issues, projects, cycles, comments, activity) through the owning modules' services.

Schema footprint:

| Change | Purpose | Formalized by |
|---|---|---|
| `mcp_token` table | One row per issued agent credential: owner, workspace, hash, label, scopes, expiry, revocation, last use | **MCP (this milestone)** |
| `McpTokenScope` enum | The closed set of permissions a token may carry | **MCP** |

Nothing else: no session table (the protocol has no sessions — ADR-005), no connection state, no per-tool state, no tool-call audit table (D9), and no denormalized mirror of membership or roles.

---

## 2. Core schema (Prisma-owned)

### 2.1 `mcp_token`

```prisma
// ── MCP Server ──────────────────────────────────────────────────────────────
// Prisma-owned (hand-modeled, like all prior features). See
// shipyard-design/04-Engineering/features/mcp/data-model.md §2.
//
// D1: a dedicated table, not columns on `user` (Better Auth-owned — F11's
// isolation principle). D2: only the SHA-256 hash of the token is stored; the
// plaintext exists once, in the creation response. D3: a token is bound to
// exactly one workspace; there is no workspace argument anywhere in the tool
// surface. D5: revocation is a timestamp — rows are never deleted.

enum McpTokenScope {
  READ           // all read tools
  ISSUES_WRITE   // create / update / status / assign / block
  COMMENTS_WRITE // add a comment
  ISSUES_DELETE  // archive / restore / delete (phase 3)
}

model McpToken {
  id          String         @id @default(cuid())
  userId      String
  workspaceId String
  label       String         @db.VarChar(60)
  tokenHash   String         @unique
  // D8: the first characters of the token, kept for recognition in the UI
  // ("shp_9f2c…") and for log redaction. Never the full token.
  tokenPrefix String         @db.VarChar(12)
  scopes      McpTokenScope[]
  expiresAt   DateTime?
  lastUsedAt  DateTime?
  revokedAt   DateTime?
  createdAt   DateTime       @default(now())
  updatedAt   DateTime       @updatedAt

  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([workspaceId])
  @@index([workspaceId, revokedAt])
  @@map("mcp_token")
}
```

`User` and `Workspace` each gain one additive relation field (`mcpTokens McpToken[]`) — the only touch on existing models, and the same additive pattern every prior feature used.

### 2.2 Column notes

| Column | Type | Notes |
|---|---|---|
| `id` | `String` (cuid) | Immutable internal id; never travels to an agent |
| `userId` | `String` | Token owner. Opaque Better Auth id (not a cuid) — validated by existence, not format |
| `workspaceId` | `String` | The single workspace this token may touch (D3) |
| `label` | `VarChar(60)` | Member-supplied name ("my laptop"); trimmed, bounded, non-unique |
| `tokenHash` | `String` unique | SHA-256 of the presented token (D2). The lookup key on every request |
| `tokenPrefix` | `VarChar(12)` | Display/redaction aid (D8) |
| `scopes` | `McpTokenScope[]` | Closed set (D4). Empty array is invalid at the service layer — a token that can do nothing is a mistake, not a state |
| `expiresAt` | `DateTime?` | `NULL` = no expiry; an expired row is treated exactly like a revoked one at auth time |
| `lastUsedAt` | `DateTime?` | Best-effort, written at most once per minute per token (D6) |
| `revokedAt` | `DateTime?` | `NULL` = active (D5) |
| `createdAt` / `updatedAt` | `DateTime` | Standard |

### 2.3 Indexes & query paths

| Index | Serves |
|---|---|
| `tokenHash` (`@unique`) | The hot path: every MCP request resolves the credential by hash in one index lookup |
| `(userId)` | "My agent connections" listing in account settings |
| `(workspaceId)` | Owner/Admin view of a workspace's tokens; cascade deletes |
| `(workspaceId, revokedAt)` | Active-token counts for a workspace (usage reporting, future limits) |

No functional or partial indexes are required — unlike F3/F4/F7 there is no case-insensitive uniqueness and no "one active row" rule to enforce in SQL.

### 2.4 Cascade contract

| Parent deleted | Effect on `mcp_token` | Rationale |
|---|---|---|
| `user` | `Cascade` | A credential with no owner is unusable; leaving it would be a live key to a deleted person's powers |
| `workspace` | `Cascade` | Workspace deletion already cascades its domain data (F2 contract); credentials are no exception |

The token row is never the parent of anything, so no other table changes behaviour because of this feature.

### 2.5 Deliberately not stored

| Not stored | Why |
|---|---|
| Plaintext tokens | D2 — a database leak must not be a credential leak |
| Sessions / connection state | The protocol removed sessions (ADR-005); every request is self-contained |
| Tool registry rows | The registry is code, versioned with the server, not data (D1 of api-design) |
| Tool-call log | D9 — Pino/Sentry provide the MVP evidence; a durable audit table needs a stated need first |
| Activity provenance column | D8 — attribution rides in the existing frozen summary text until evidence justifies a column |
| A mirror of role or membership | Role is read live from `workspace_member` on every request; a cached copy would be a stale-permission bug |

---

## 3. Key decisions & alternatives

### D1 — A dedicated table, not columns on `user`

**Decision:** `mcp_token` — its own table with a relation to the owner. *Rejected:* a `token`/`mcpToken` column on `user` (single-token limit, no labels, no per-token scopes/expiry, and it edits a Better Auth-generated table), and a workspace-column variant on `workspace_member` (mixes membership semantics with credentials; one token per member per workspace, which is too strict and still can't carry labels or independent revocations).

### D2 — Hash at rest, plaintext shown once

**Decision:** store SHA-256 of the token; return the token exactly once, in the creation response. *Rationale:* the token is 256 bits of CSPRNG output, so there is no guessable input space to attack offline — a fast hash is sufficient and keeps the per-request lookup cheap. This is deliberately **not** the password rule (bcrypt/argon2) and the difference is worth stating: a password is low-entropy and guessable, a random token is not. If a future change ever lets members *choose* a token value, this decision must be revisited together with a slow KDF. *Rejected:* encrypting tokens at rest (reversible, so a key leak is still a total compromise) and storing plaintext (a database leak becomes a fleet of live keys).

### D3 — Workspace-bound tokens, no workspace argument

**Decision:** every token names exactly one `workspaceId`, and no tool accepts a workspace parameter. *Rationale:* the workspace becomes a property of the credential, so there is nothing for a caller to forge, nothing to validate per call, and no path by which an agent can address a workspace its owner never authorized. *Rejected:* a token plus a `workspace` argument (every call needs a membership check, and the argument is untrusted input by construction); a single token spanning all of a member's workspaces (wider blast radius for a stolen credential, and the audience-binding requirement of the eventual OAuth flow is per-resource anyway).

### D4 — Scopes as a Postgres enum array, not a join table

**Decision:** `McpTokenScope[]` on the token row. *Rationale:* the set is small, fixed, and known at compile time; it is read on every request and filtered against the tool registry in memory; an array keeps both the write path (create/update) and the read path single-row. *Rejected:* a `mcp_token_scope` join table (a join on the hottest path for a set that will never exceed a handful of values) and free-text scopes (no compile-time safety, no guarantee that an unknown scope is rejected).

### D5 — Revocation is a timestamp, not a deleted row

**Decision:** `revokedAt` set; the row stays. *Rationale:* the UI must be able to show "revoked on …" and the last-used time that preceded it; a deleted row destroys the evidence a member needs when deciding whether a leaked token was ever used. Filtering active tokens is `revokedAt: null`. *Rejected:* hard deletion (loses the audit trail, and a revoke-then-recreate cycle silently loses history).

### D6 — `lastUsedAt` is best-effort, throttled to once per minute

**Decision:** the auth path updates `lastUsedAt` only when the stored value is `NULL` or older than one minute, and the update is not part of the tool's transaction. *Rationale:* a write per tool call doubles the write load of a read-only agent, and the field is for anomaly detection ("was this token used after I stopped using it?"), where minute precision is plenty. *Rejected:* exact per-call tracking (writes on every read) and a separate usage table (D9 reasoning: no stated need yet).

### D7 — Expiry optional at creation, absolute at auth time

**Decision:** `expiresAt` may be `NULL` (no expiry, the default for self-hosted simplicity) and an expired token is treated exactly like a revoked one — `401`, same message. *Rationale:* members who want rotation get it; members who don't aren't forced into a lifecycle they'll work around by re-creating tokens. The auth path has exactly one "is this usable?" predicate, so there is no second code path to keep correct. *Rejected:* mandatory short expiries (incompatible with an unattended agent that must run overnight or on a server) and a refresh mechanism (that is the OAuth milestone's job, not this one's).

### D8 — No provenance column on the activity log; attribution rides in the summary

**Decision:** the activity log keeps recording the **member** as the actor, and the MCP write path includes the agent provenance in the existing frozen `summary` text. *Rationale:* `actorName` is an identity field — putting "Ada (agent)" in it would corrupt every future query that groups by actor, and F-activity deliberately treats the summary as the human-readable record. *Rejected:* a new `activity_event.source` column now (a schema change to a shipped feature for an MVP need that text satisfies) — recorded here as **deferred with a stated trigger**: if the product ever needs to *filter or report* by provenance, add the column then, with a backfill of unknown for existing rows.

### D9 — No tool-call audit table in the MVP

**Decision:** tool calls are logged (Pino) with tool name, argument keys, duration, result size, and error state; nothing is persisted to Postgres. *Rationale:* the MVP needs evidence to improve tool descriptions and to answer "which tools are used", and structured logs answer both. A durable table would be a new write path on every call, with its own retention question, before anyone has asked for a per-call audit. *Trigger to revisit:* a member-visible "agent activity" view, or a compliance/retention requirement.

---

## 4. Invariants

1. `tokenHash` is unique — two rows can never answer to the same presented token.
2. A token resolves to exactly one `(userId, workspaceId)` pair, and that pair must still be a live membership at request time; a removed member's tokens stop working immediately (membership is the second door, always checked live).
3. Scopes are a **subset** of what the member may do — enforced at issuance (a member cannot mint a token whose scopes exceed their role's abilities) and again at use (the role check runs per action regardless of scope).
4. Revoked or expired ⇒ indistinguishable from invalid at the transport layer (`401`, one message).
5. The plaintext token exists only in the creation response; it is never selectable from the database and never logged.
6. Deleting a workspace or a user removes their tokens by cascade — no orphaned credentials.

---

## 5. Lifecycle & state matrix

| State | How reached | Behaviour at auth | UI |
|---|---|---|---|
| **Active** | Created | Accepted (subject to scopes + membership) | Listed with label, scopes, created/last-used |
| **Expired** | `expiresAt` in the past | `401` (same as revoked) | Listed, marked expired, offer "create a new one" |
| **Revoked** | Member revokes; or owner removes it | `401` | Listed, marked revoked, shows `revokedAt` |

Transitions: `Active → Revoked` (irreversible, D5) and `Active → Expired` (time-based, D7). There is no un-revoke: a new credential is created instead, which keeps the history honest.

---

## 6. Where the rest of the feature's state lives

| Concern | Provided by | Note |
|---|---|---|
| Workspace, membership, role | `workspace` / `members` tables | Read live per request; never mirrored |
| Issue/project/cycle/comment data | Their owning tables | Reached only through the owning services |
| Activity events for agent writes | `activity_event` | Written by the owning services inside their transactions — that is the attribution record (D8) |
| Notifications triggered by agent actions | `notification` | Emitted by the owning services exactly as in the UI (no MCP-specific recipients) |
| Tool descriptions, schemas, annotations | Code (`features/mcp/registry.ts` + shared contracts) | Versioned with the server; not data |

---

## 7. Deferred schema work

| Item | Trigger |
|---|---|
| `activity_event.source` provenance column | A need to filter/report agent-performed activity (D8) |
| `mcp_tool_call` audit table | A member-visible agent-activity view, or retention requirements (D9) |
| OAuth client/consent/token tables | The OAuth milestone (spec §7 Q1) |
| Per-tool grants beyond scopes | Evidence that scopes are too coarse for real users |
| Workspace-level token limits/quotas | Usage data from logs showing a need |
