# Information Architecture

## Overview

This document defines how information is organized within Shipyard. It establishes the hierarchy of the product, the relationships between resources, and the logical grouping of functionality.

The Information Architecture serves as the foundation for navigation, screen design, user flows, and implementation.

---

# Architecture Principles

The information architecture follows these principles:

- Workspace-first organization
- Clear separation of features
- Predictable hierarchy
- Minimal navigation depth
- Consistent resource relationships
- Scalable for future features

---

# Product Hierarchy

```text
Shipyard
└── User Account
    ├── Workspace
    │   ├── Dashboard
    │   ├── Issues
    │   ├── Projects
    │   ├── Cycles
    │   ├── Members
    │   ├── Notifications
    │   └── Settings
    └── Workspace Switcher
```

---

# Resource Hierarchy

```text
User
│
├── Workspaces
│   ├── Members
│   ├── Projects
│   │   └── Issues
│   ├── Cycles
│   │   └── Issues
│   ├── Notifications
│   └── Settings
```

---

# Primary Sections

## Dashboard

The landing page for a workspace that provides a high-level overview of personal work, team activity, active projects, and development cycles.

---

## Issues

The central workspace for creating, organizing, tracking, and completing development work.

Contains:

- My Issues
- All Issues
- Backlog
- Archived Issues

---

## Projects

Groups related issues into larger initiatives and provides progress tracking across the project lifecycle.

Contains:

- Active Projects
- Planned Projects
- Completed Projects
- Archived Projects

---

## Cycles

Organizes work into fixed development iterations and tracks progress toward cycle goals.

Contains:

- Active Cycle
- Planned Cycles
- Completed Cycles
- Archived Cycles

---

## Members

Manages workspace membership, roles, and permissions.

Contains:

- Member List
- Invitations
- Roles

---

## Notifications

Displays user-specific activity such as mentions, assignments, comments, and system updates.

Contains:

- Unread Notifications
- Read Notifications

---

## Settings

Allows users and workspace administrators to configure personal preferences and workspace settings.

Contains:

- Profile
- Preferences
- Workspace Settings
- Member Management
- Danger Zone

---

# Resource Relationships

```text
Workspace
├── Members
├── Projects
│   └── Issues
├── Cycles
│   └── Issues
├── Notifications
└── Settings
```

Relationship Rules

- Every Project belongs to one Workspace.
- Every Issue belongs to one Workspace.
- Every Issue may belong to one Project.
- Every Issue may belong to one Cycle.
- Every Cycle belongs to one Workspace.
- Every Member belongs to one Workspace.
- Every Notification belongs to one User within a Workspace.

---

# Information Ownership

| Resource | Parent |
|-----------|--------|
| Workspace | User |
| Member | Workspace |
| Project | Workspace |
| Issue | Workspace |
| Cycle | Workspace |
| Notification | User |
| Settings | Workspace / User |

---

# Future Expansion

The architecture is designed to support future modules without restructuring the product hierarchy.

Potential future sections include:

- Roadmaps
- Calendar
- Timeline
- Integrations
- Analytics
- Templates
- AI Features
- Automation