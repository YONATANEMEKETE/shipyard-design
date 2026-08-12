# Shipyard Design & Plan

The official **design and planning repository** for [**Shipyard**](https://github.com/YONATANEMEKETE/shipyard) — *Plan. Build. Ship.* — an open-source project management platform for small software engineering teams.

This repository contains all product documentation, UX artifacts, UI designs, technical planning, and architecture documents created before implementation. It serves as the single source of truth for product decisions and provides a complete history of how Shipyard evolves from an idea into a production-ready application.

The application source code is maintained separately in the [**shipyard**](https://github.com/YONATANEMEKETE/shipyard) repository — created when engineering kicks off.

**Status (2026-08-12):** Product ✅ · UX ✅ · UI ✅ (Harbor Amber design system finalized, shadcn v4 theme exported to `03-ui/exports/`) · Engineering ⏳ (next: `04-engineering/`)

---

## Repository Structure

```text
shipyard-design/
├── 01-product/
│   ├── Product Brief.md
│   └── PRD.md
│
├── 02-ux/
│   ├── User Flows.md
│   ├── Information Architecture.md
│   ├── Wireframes/
│   └── UX Decisions.md
│
├── 03-ui/
│   ├── pencil/
│   ├── exports/          ← shadcn v4 / Tailwind v4 theme export (globals.css)
│   ├── Design System.md
│   └── Components.md
│
├── 04-engineering/
│   ├── Implementation Plan.md
│   ├── Architecture.md
│   ├── Database Schema.md
│   ├── API Specification.md
│   └── Technical Decisions.md
│
├── assets/
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
6. Technical Planning
7. Engineering Handoff
8. Implementation (Shipyard Repository)

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