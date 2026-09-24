# Shipyard Design & Plan

The official **design and planning repository** for [**Shipyard**](https://github.com/YONATANEMEKETE/shipyard) — *Plan. Build. Ship.* — an open-source project management platform for small software engineering teams.

This repository contains all product documentation, UX artifacts, UI designs, technical planning, and architecture documents. It is the single source of truth for product decisions and provides a complete history of how Shipyard evolved from an idea into a production application.

The application source code and its deployment configuration are maintained separately in the [**shipyard**](https://github.com/YONATANEMEKETE/shipyard) repository. Local setup and self-hosting instructions live in that repository's README.

|                 |                                              |
| --------------- | -------------------------------------------- |
| **Live app**    | <https://shipyard.yonatanem.com>             |
| **API**         | <https://api.shipyard.yonatanem.com>         |
| **Source code** | <https://github.com/YONATANEMEKETE/shipyard> |

---

## Status — in production (2026-09-24)

**Product ✅ · UX ✅ · UI ✅ · Engineering ✅ — the MVP is implemented and deployed; launch preparation is in progress.**

| Area                                  | State                                                                                                                                                                      |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Product, UX, UI                       | Complete — Harbor Amber design system finalized, shadcn v4 / Tailwind v4 theme exported to `03-UI/exports/`                                                                |
| MVP features (F1–F11)                 | Implemented in the `shipyard` repository against the specs in `04-Engineering/features/*`, each as a vertical slice                                                        |
| F12 — hardening and release readiness | Implemented — runtime hardening, the public web/API split, the API production image, and the operational runbook (`deployment.md`)                                         |
| F13 — MCP server                      | Implemented — the first post-MVP feature, shipped as `/mcp` on the API origin (ADR-005)                                                                                    |
| Deployment                            | **Live** — Vercel (web) and Render (API, Docker) with Neon Postgres, Cloudflare R2, and Resend, per ADR-006 and ADR-007                                                    |
| Observability                         | Sentry (errors), PostHog (product analytics and Web Vitals), and OpenTelemetry → Grafana Cloud (traces, metrics, logs)                                                     |
| Self-hosting                          | The API image, the local compose file, and the required environment are documented in the `shipyard` repository's README                                                   |
| Open                                  | Launch readiness — the final smoke pass against production and launch preparation. Nothing here is a feature gap; post-MVP candidates stay unscheduled in `05-Post-MVP.md` |

---

## Where to find things

| Question                                         | Document                                                     |
| ------------------------------------------------ | ------------------------------------------------------------ |
| What are we building, and for whom?              | `01-Product/` — Product Brief and PRD                        |
| How should a flow behave?                        | `02-UX/` — flows, screen inventory, navigation, empty states |
| How should it look?                              | `03-UI/design.md` and `03-UI/shipyard.pen`                   |
| What does a feature do?                          | `04-Engineering/features/<feature>/spec.md`                  |
| How is that feature built?                       | `…/data-model.md` and `…/api-design.md` beside its spec      |
| Why was a technical decision made?               | `04-Engineering/adr/`                                        |
| How does the system fit together?                | `04-Engineering/00-architecture.md`                          |
| What runs in production, and how is it operated? | `04-Engineering/deployment.md`                               |
| In what order was it built?                      | `04-Engineering/implementation/Implementation Plan.md`       |
| What comes after the MVP?                        | `05-Post-MVP.md`                                             |

---

## Repository Structure

```text
shipyard-design/
├── 01-Product/
│   ├── Product Brief - Shipyard.md
│   └── Shipyard Product Requirements Document (PRD).md
│
├── 02-UX/
│   ├── UserFlows.md · Information-Archtecture.md · Navigations.md
│   ├── Screen-Inventory.md · Empty-states.md · UX-decisions.md
│   ├── User-Personas.md
│   └── wireframes/
│
├── 03-UI/
│   ├── design.md                       ← Harbor Amber design system
│   ├── shipyard.pen                    ← UI design (Pencil)
│   ├── shipyard-logo-system.pen        ← logo and social identity source
│   ├── exports/                        ← shadcn v4 / Tailwind v4 theme export
│   └── references/ · Inpos/
│
├── 04-Engineering/
│   ├── 00-architecture.md              ← high-level system architecture
│   ├── deployment.md                   ← the running system: topology, the
│   │                                       environment matrix, health and
│   │                                       observability, backups, rollback
│   ├── adr/                            ← ADR-001 … ADR-007: stack, repo
│   │                                       layout, web↔api, infrastructure,
│   │                                       the MCP surface, the public
│   │                                       split, and Vercel + Render
│   ├── features/
│   │   └── <feature>/                  ← spec.md (behavior only) plus the
│   │                                       technical design produced at
│   │                                       implementation time:
│   │                                       data-model.md · api-design.md
│   └── implementation/
│       └── Implementation Plan.md      ← ordered milestones (F0–F13) and the
│                                            per-feature implementation loop
│
├── 05-Post-MVP.md                      ← living backlog after the MVP release
└── README.md
```

---

## Design Workflow

Every major feature follows the same planning process:

1. Product Discovery
2. Product Brief
3. Product Requirements Document (PRD)
4. UX Planning
5. UI Design
6. Feature Spec (behavior only — `04-Engineering/features/<feature>/spec.md`)
7. Technical Design — the domain model, data model, and API design are written per feature, beside its spec in this repository, before implementation starts
8. Implementation — the `shipyard` repository builds the slice against that design, and the documents here are corrected when reality diverges

---

## Purpose

This repository exists to:

- Document product decisions
- Plan features before implementation
- Maintain UX and UI artifacts
- Define the technical architecture
- Record design and engineering decisions
- Create a clear handoff for development

---

## Related Repository

| Repository                                                               | Purpose                                                               |
| ------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| [**shipyard-design**](https://github.com/YONATANEMEKETE/shipyard-design) | Product planning, UX, UI, architecture, and documentation — this repo |
| [**shipyard**](https://github.com/YONATANEMEKETE/shipyard)               | Application source code, deployment configuration, and self-hosting   |

---

## Philosophy

Plan first. Build second.

Every implementation should be backed by clear product requirements, thoughtful UX, consistent UI, and documented technical decisions.
