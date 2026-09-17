# MCP Server — Feature Spec

**Status:** Draft for review
**Last updated:** 2026-09-17
**Design sources:** `05-Post-MVP.md` §First after MVP (MCP server — remote Streamable HTTP, user-scoped tokens, read tools first, non-destructive writes second, agent attribution) · `01-Product/PRD.md` (member capabilities, permission matrix, archive ≠ delete, workspace isolation) · `03-UI/design.md` (account settings surface for token management)
**Technical design:** Produced during this feature's implementation step — `features/mcp/data-model.md`, `features/mcp/api-design.md`, and `adr/ADR-005-mcp-server-surface.md`.

---

## 1. What this feature is about

MCP Server lets a member's **AI agent work inside Shipyard on that member's behalf**: finding and reading work, then creating and updating it, through the same services, rules, and permissions the web app uses.

The feature is a **second interface to the same domain**, not a second backend. It adds no business rules of its own: every action an agent takes is the action of the member who authorized it, passes the same validation, role checks, and transactions, and appears in the product's records (history, activity, notifications) exactly as if that member had done it in the UI.

Delivery is deliberately staged — **read first, additive writes second, destructive last** — so the surface (the set of things an agent can do and what it can see) is proven useful before it is given the ability to change data.

## 2. What users can do

Through their agent, a member can:

- **Connect an agent** to a workspace by creating a token; see their tokens, when each was last used, and revoke any of them.
- **Find work:** browse and filter issues (status, priority, assignee, project, cycle, label, blocked, due dates, archived), read a single issue in full including its recent history, and search across issues, projects, cycles, members, and comments when the identifier is unknown.
- **See the shape of things:** the workspace overview (their work, active projects, the current cycle, blocked and overdue items, recent activity), the projects list, the cycles list, and the member directory.
- **Act on work:** create an issue, update its title/description/priority/due date/project, move it between statuses, assign or unassign it, block or unblock it, and add a comment.
- **Remove work (gated):** archive, restore, and permanently delete an issue — only when the member's role and the token's permissions allow it, and only with explicit human confirmation.

## 3. Main behaviors & actions

### 3.1 Connection & identity

- A token belongs to **exactly one workspace** and to **one member**. There is no workspace argument anywhere: the workspace is a property of the credential, never of a request parameter.
- A token carries **permissions (scopes)** that are a subset of what its owner may do — never more. A read-only token is the default.
- The member's **workspace role** (Owner / Admin / Member) still governs every action on top of the token's scopes. Tokens can never widen what a person may do.
- Tokens are shown **once** at creation, are identified by a label, expire if an expiry was set, and can be revoked at any time. Revocation takes effect on the next request.
- An agent authenticates with a token only. Session cookies are never accepted on the MCP surface.

### 3.2 Read behavior

- Results are **bounded**: list tools return a capped number of compact rows, and always report when they truncated (how many returned of how many matched) so the agent never reports a partial set as complete.
- **Archived work is excluded by default** and included only when explicitly requested — matching the active-work bias of the UI.
- Identifiers are the product's own (`SHIP-###`): a member can name work the way they see it, and the system resolves it.
- Single-entity reads return the entity's detail (description, creator, labels, assignment, project, cycle) plus recent history; list reads return compact cards only.
- Reads never leak across workspaces: something that does not exist and something the caller may not see are **indistinguishable**.

### 3.3 Write behavior

- Every write goes through the owning module's existing service — the same validation, invariants, transactions, and side effects as the UI. Agents get no shortcut and no bypass.
- Consequences that already exist in the product therefore apply automatically: status transitions set and clear blocked state correctly, assignment changes emit notifications, every change writes its history row, and every action lands in the activity log.
- "Clear a value" is expressible (unassign, detach from a project, detach from a cycle), and the tool description says so explicitly, because the difference between "leave as is" and "clear it" is not something an agent can guess.
- Cycles cannot be set at issue creation (a product rule) — the tool says so and directs the agent to create first, then update.

### 3.4 Destructive behavior

- Permanently deleting data always requires, in this order: the token's delete permission, an Owner/Admin role, and **human confirmation** — the person, not the model. A model-typed confirmation is never accepted as consent.
- Archive and restore are reversible and are treated as ordinary gated actions, not as deletions.
- Bulk destructive operations are not offered at all.

### 3.5 Error behavior

- A domain failure (validation, permission, archived resource, conflict, missing resource) is returned as a **readable failure the agent can act on**: what failed, which item, and what to do next. Codes alone are never the whole message.
- Transport-level problems (bad credential, unsupported protocol version, malformed request) fail at the transport level and are not surfaced as work results.
- Failures never disclose anything the caller may not see, and never echo credentials.

### 3.6 Attribution

- Work performed through an agent is recorded as the **member's** action, with the agent provenance visible — so a reader of the activity log can always tell that it was the member's agent rather than the member in the UI.
- Notifications address the same recipients they would in the UI; an agent's action never notifies a different audience.

### 3.7 Visible surface

- A token only ever sees the tools its scopes permit. A read-only token's agent literally does not discover write tools, so it cannot attempt them by mistake.

## 4. User flows (high level)

1. **Connect:** account settings → create a token for a workspace (read-only by default) → copy it once → paste into the agent → the agent is working in that workspace.
2. **Ask a question:** "what's blocked in the current cycle?", "find anything about login", "what did the team do yesterday?", "who's working on what?" — answered by reads, with no tool named by the user.
3. **Create work:** "file a bug for the login redirect, high priority, assign it to me" — the agent creates the issue, and it appears in the product like any other issue, attributed.
4. **Update work:** "move SHIP-42 to in progress and leave a note" — status change plus comment, each recorded.
5. **Remove work (gated):** "delete SHIP-42" → the agent reports that a human must confirm; confirmation happens in the person's own interface.
6. **Revoke:** account settings → see last-used times → revoke a token → the agent stops working immediately.

## 5. Business rules

1. A token belongs to one workspace and one member; there is no workspace parameter on any action.
2. Every action passes the same validation, invariants, transactions, and permissions as the equivalent UI action — no agent shortcut.
3. Tokens can only narrow a member's abilities (scopes + role), never widen them.
4. Reads are bounded, and truncation is always reported.
5. Archived resources are excluded by default and readable only when explicitly requested.
6. Archived resources are read-only: writes against them fail with a clear, actionable message.
7. A non-member and a non-existent resource are indistinguishable in every response.
8. Destructive actions require the delete permission, an Owner/Admin role, and explicit human confirmation.
9. Model-supplied "confirmations" are never treated as human consent.
10. Agent actions are attributed to the member, with provenance recorded.
11. Credentials are never logged, never returned after creation, and stored only as a hash.
12. Revocation and expiry take effect on the next request.
13. Account-level and membership-level changes (profile, password, email, invitations, roles, ownership, workspace lifecycle) are not exposed to agents.
14. The MCP surface adds no new business rules to the product; where a rule does not exist, it is added to the owning module's service, not to the agent layer.

## 6. Out of scope (MVP)

Invitations, role changes, ownership transfer, workspace create/archive/restore/delete, account settings (profile, avatar, password, email), notification management, billing, bulk operations, cross-workspace tokens, attachments/files, realtime streams and push, a generic "call any endpoint" capability, per-tool permissioning beyond scopes, and OAuth (deferred to a later milestone; personal access tokens are the MVP mechanism).

## 7. Open product questions

| # | Question | Notes |
|---|---|---|
| 1 | OAuth timing | Post-MVP milestone; PATs ship first. Confirm no external/third-party agent connections are required in v1 |
| 2 | Member email on the agent surface | Assignment needs a name, not an address — recommend excluding email from agent-visible member fields |
| 3 | Attribution granularity | Provenance in the activity summary vs a dedicated source column on the activity log — decide before phase 3 |
| 4 | Delete confirmation mechanics | Host consent (agent client prompts) vs an explicit in-protocol confirmation round-trip — decide before phase 3 |
| 5 | Multi-workspace users | One token per workspace (recommended for v1) vs a token that spans workspaces — revisit with evidence |
| 6 | Tool-call auditing | Whether performed tool calls are worth a durable audit table, or Pino/Sentry logging suffices |
