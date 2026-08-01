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
    ├── Account Settings
    │   ├── Profile
    │   ├── Security
    │   └── Appearance
    ├── Workspace
    │   ├── Dashboard
    │   ├── Issues
    │   ├── Projects
    │   ├── Cycles
    │   ├── Members
    │   ├── Notifications
    │   └── Workspace Settings
    └── Workspace Switcher
        ├── Active Workspaces
        └── Archived Workspaces (Owner only)
            └── Archived Workspace Summary
                ├── Restore Workspace
                └── Delete Workspace
```

---

# Resource Hierarchy

```text
User
│
├── Account Settings
└── Workspaces
    ├── Members
    ├── Projects
    │   └── Issues
    ├── Cycles
    │   └── Issues
    ├── Notifications
    └── Workspace Settings
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
- Blocked Issues
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

Non-archived Cycles form a sequential workspace schedule: their date ranges never overlap, and no more than one Cycle can be Active.

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

## Account Settings

Allows an authenticated user to manage account-level information without requiring an active workspace.

Contains:

- Profile
- Security
- Appearance

---

## Workspace Settings

Allows the Workspace Owner and other members with specific permissions to view or manage settings for the active workspace.

Contains:

- Workspace Details
- Member Management
- Danger Zone

The active Workspace Danger Zone provides Archive. Permanent deletion is available only from the Archived Workspace Summary after archival.

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
└── Workspace Settings
```

Relationship Rules

- Every Workspace has an immutable internal identifier; its display name may duplicate another Workspace name.
- Navigation, permissions, invitations, and resource relationships reference the Workspace identifier rather than its display name.
- Every Project belongs to one Workspace.
- Every Project has exactly one Project Owner who is a current member of that Workspace; Project ownership represents accountability and grants no additional permissions.
- Every non-deleted Project has a name unique within its Workspace; Archived Projects continue to reserve their names.
- Every Issue belongs to one Workspace.
- Every Issue may belong to one Project.
- Every Issue may belong to one Cycle.
- Every Cycle belongs to one Workspace.
- Every non-deleted Cycle has a name unique within its Workspace; Archived Cycles continue to reserve their names.
- Every Member belongs to one Workspace.
- Every Notification belongs to one User within a Workspace.
- Projects and Cycles are independent and have no direct ownership relationship.
- A Project may show Cycles represented by its Issues, but that relationship is derived through those Issues.

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
| Account Settings | User |
| Workspace Settings | Workspace |

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
