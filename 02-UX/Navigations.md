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
└── Workspace
    ├── Dashboard
    ├── Issues
    ├── Projects
    ├── Cycles
    ├── Members
    ├── Notifications
    └── Settings
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
- Settings

---

# Secondary Navigation

Some sections provide local navigation for related views.

### Issues

- My Issues
- All Issues
- Backlog
- Archived

### Projects

- Active
- Planned
- Completed
- Archived

### Cycles

- Active
- Planned
- Completed
- Archived

### Settings

- Profile
- Preferences
- Workspace
- Members
- Danger Zone

---

# Global Navigation Elements

Available from any page:

- Workspace Switcher
- Global Search
- Create Button
- User Menu
- Notifications

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
- Member Profile

Users should be able to bookmark or share links to individual resources.

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
