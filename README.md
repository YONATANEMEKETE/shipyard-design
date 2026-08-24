# Shipyard Design & Plan

The official **design and planning repository** for [**Shipyard**](https://github.com/YONATANEMEKETE/shipyard) — *Plan. Build. Ship.* — an open-source project management platform for small software engineering teams.

This repository contains all product documentation, UX artifacts, UI designs, technical planning, and architecture documents created before implementation. It serves as the single source of truth for product decisions and provides a complete history of how Shipyard evolves from an idea into a production-ready application.

The application source code is maintained separately in the [**shipyard**](https://github.com/YONATANEMEKETE/shipyard) repository — created when engineering kicks off.

**Status (2026-08-22):** Product ✅ · UX ✅ · UI ✅ (Harbor Amber design system finalized, shadcn v4 theme exported to `03-UI/exports/`) · Engineering ⏳ (per-feature behavior specs in `04-engineering/features/*/spec.md`; technical design is produced per feature at implementation time) — next: F1 Auth (Implementation Plan)

---

## Repository Structure

```text
shipyard-design/
├── 01-Product/
│   ├── Product Brief.md
│   └── PRD.md
│
├── 02-UX/
│   ├── UserFlows.md
│   ├── Information-Archtecture.md
│   ├── Screen-Inventory.md
│   ├── Navigations.md
│   ├── UX-decisions.md
│   ├── Empty-states.md
│   ├── User-Personas.md
│   └── wireframes/
│
├── 03-UI/
│   ├── design.md (Design System)
│   ├── shipyard.pen                    ← UI design (Pencil)
│   ├── exports/                        ← shadcn v4 / Tailwind v4 theme export
│   └── references/ · Inpos/
│
├── 04-Engineering/
│   ├── 00-architecture.md              ← high-level system architecture
│   ├── adr/                            ← architecture decision records (stack, mono- repo, web↔api, deployment)
│   ├── features/
│   │   └── <feature>/spec.md           ← BEHAVIOR-ONLY feature specs: what the
│   │                                       feature is about, user capabilities,
│   │                                       main behaviors, user flows, business
│   │                                       rules. No data model / API design —
│   │                                       technical design is produced per
│   │                                       feature at implementation time.
│   └── implementation/
│       └── Implementation Plan.md      ← ordered milestones (F0–F12) + the
│                                            per-feature implementation loop
│
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
6. Feature Spec (behavior only — `04-engineering/features/<feature>/spec.md`)
7. Implementation — each feature starts with its own technical planning (domain model, data model, API design, system design), driven by the feature spec and recorded in the shipyard repository

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

| Repository | Purpose |
|------------|---------|
| [**shipyard-design**](https://github.com/YONATANEMEKETE/shipyard-design) | Product planning, UX, UI, architecture, and documentation — this repo |
| [**shipyard**](https://github.com/YONATANEMEKETE/shipyard) | Application source code and implementation — created at engineering kickoff |

---

## Philosophy

Plan first. Build second.

Every implementation should be backed by clear product requirements, thoughtful UX, consistent UI, and documented technical decisions.