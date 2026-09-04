# Post-MVP Roadmap

**Status:** Living backlog — candidates for after the MVP release, not commitments
**Last updated:** 2026-09-04
**Sources:** `Out of scope` / deferred / handoff sections across `04-Engineering/features/*` (auth → settings, plus activity), `00-architecture.md`, `Implementation Plan.md` §7, and design discussions 2026-09-04 (MCP, Activity Log).

> Rule for promoting anything below into the MVP or the first post-MVP milestone: evidence first (user reports, measured pain, or a blocking dependency) — never preemptive building.

---

## First after MVP (positioned, not yet scheduled)

- **MCP server** — remote Streamable HTTP endpoint, user-scoped tokens (PATs → OAuth), read tools first, non-destructive writes second, agent attribution in the activity log. Needs `features/mcp/` spec (spec → data-model → api-design, per repo flow).
- **Activity Log feed swap** — dashboard Recent Activity migrates onto the activity log when it lands (recorded F9 handoff). MVP-adjacent; do it before any analytics work that would otherwise fork feed logic.

---

## Issues & planning

- Recurring cycles; carry-over of unfinished issues into the next cycle
- Velocity reporting, burndown charts, capacity planning, workload balancing
- Parent-child issues, dependencies, subtasks, custom issue types, issue templates
- Watchers, story points, manual card ordering/rank
- Per-workspace identifier prefixes (beyond constant `SHIP-`)
- Rich-text editing; file attachments on issues

## Discussion & attention

- Comment threads/replies, emoji reactions, comment pinning, edit history/diffs
- Code blocks with syntax highlighting; file attachments on comments
- Notification preferences/opt-outs, grouping, snoozing, digests
- Email/push/desktop delivery; realtime push (WebSocket)
- Notifications for unassignment, status changes, invitations, ownership, cycle events

## Discovery & insights

- Dashboard: customizable layouts, widgets, drag-to-reorder, pinned widgets
- Analytics, reports, workload charts, cross-workspace dashboard
- Search: advanced operators, fuzzy/typo tolerance (`pg_trgm`), result highlighting/snippets, saved views, cross-workspace search, Meilisearch migration
- Activity Log: CSV export, retention jobs, realtime updates, backfill of pre-feature history

## Accounts & workspace

- Custom roles / permission editor, team groups, bulk member import, SSO/SCIM
- Per-workspace themes, profile visibility settings, locale/language settings
- Custom avatar presets, animated avatars, server-side image processing
- Member count limits / quotas; account deletion flow
- Invitation token hashing at rest; user-handle column (revisit if mention-ambiguity reports arrive)

## Platform & scale

- Queues + background workers (email digests, cleanup janitors); outbox pattern
- Presigned direct-to-R2 uploads; public browser-to-API access + CORS
- Mobile app (API reachability decision); multi-region deployment
- Billing and usage limits; Kubernetes/microservices (only with demonstrated need)

---

## Never without a rewrite proposal

Microservices before the monolith hurts, and external engines before measured pain. Anything in Platform & scale above requires a one-page ADR with evidence attached — not enthusiasm.
