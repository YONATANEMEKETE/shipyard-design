# Navigation

## Overview

This document defines how users navigate through Shipyard. It describes the primary navigation structure, secondary navigation patterns, global actions, and navigation principles that ensure a fast, predictable, and consistent user experience.

---

# Navigation Principles

Navigation should be:

- Simple and predictable
- Consistent across the product
- Minimize the number of clicks
- Keep users oriented within the workspace
- Prioritize frequently used actions
- Support keyboard-first workflows where appropriate

---

# Navigation Hierarchy

```text
User
├── Account Settings
└── Workspace
    ├── Dashboard
    ├── Issues
    ├── Projects
    ├── Cycles
    ├── Members
    ├── Notifications
    └── Workspace Settings
```

---

# Primary Navigation

The primary navigation is available throughout the application and provides access to the core workspace sections.

Items:

- Dashboard
- Issues
- Projects
- Cycles
- Members
- Notifications
- Workspace Settings

---

# Secondary Navigation

Some sections provide local navigation for related views.

### Issues

- My Issues *(List/Kanban toggle)*
- All Issues *(List/Kanban toggle)*
- Backlog *(List only)*
- Blocked *(List only)*
- Archived *(List only)*

### Projects

- Non-archived Projects *(List/Kanban toggle)*
- Active *(List section)*
- Planned *(List section)*
- Completed *(List section)*
- Archived *(List only)*

### Cycles

- Active
- Planned
- Completed
- Archived

---

# Resource View Toggles

- All Issues, My Issues, and non-archived Projects expose a List/Kanban segmented control.
- The last selected view is stored per user and workspace, independently for Issues and Projects.
- List is the default before the user has saved a preference for that resource and workspace.
- Navigating to a list-only subview does not overwrite the saved Kanban preference.
- Search, filters, and sorting remain active when switching views.
- Cycles and all Archived resource views remain list-only.

### Account Settings *(User Menu)*

- Profile
- Security
- Appearance

### Workspace Settings

- Workspace Details
- Danger Zone

---

# Global Navigation Elements

Available from any page:

- Workspace Switcher
- Global Search
- Create Button *(active workspace only)*
- User Menu
- Notifications

---

# Global Create Behavior

The Create Button opens a menu containing only actions available to the current user:

- **Create Issue** for Owners, Admins, and Members.
- **Create Project** for Owners and Admins.
- **Create Cycle** for Owners and Admins.

Unauthorized creation actions are hidden, not disabled. The Create Button is unavailable when no active workspace is selected or while viewing an Archived Workspace.

---

# Navigation Patterns

Users should be able to:

- Navigate between sections without losing context.
- Return to the previous view using browser navigation.
- Open resources directly from notifications or search.
- Access related resources through contextual links.

---

# Deep Linking

Every major resource should have a unique URL.

Examples:

- Workspace
- Issue
- Project
- Cycle

Users should be able to bookmark or share links to individual resources.

Member Details is a contextual drawer opened from the Members Directory or Global Search. It does not have a standalone URL and cannot be bookmarked or shared in the MVP.

---

# Navigation Behavior

- The active section is clearly highlighted.
- The current workspace is always visible.
- Navigation state persists during normal browsing.
- Switching workspaces refreshes all workspace-specific data.
- Workspace owners can view archived workspaces in the Workspace Switcher and open them for restoration.
- Unauthorized navigation redirects users appropriately.

---

# Future Enhancements

- Keyboard command palette
- Recently viewed resources
- Navigation history
- Favorites
- Pinned projects
- Custom sidebar sections
