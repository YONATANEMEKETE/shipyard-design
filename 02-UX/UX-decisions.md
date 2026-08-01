# 8. UX Decisions

## 8.1 Overview

This section documents the key UX decisions made during the design of the Shipyard MVP. These decisions capture the rationale behind the product's navigation, information architecture, interaction patterns, and user experience principles. They provide a shared reference for designers and developers, ensuring future implementation and feature expansion remain consistent with the original design intent.

---

## 8.2 Navigation Decisions

### Decision 1 — Dashboard is the Primary Landing Page

**Context**

Users need an immediate overview of their workspace after entering the application.

**Rationale**

The Dashboard provides quick access to assigned work, active projects, current cycles, recent activity, and common actions, reducing unnecessary navigation.

**Impact**

Users can immediately understand their priorities and resume work.

---

### Decision 2 — Sidebar as Primary Navigation

**Context**

Shipyard contains multiple primary modules.

**Rationale**

A persistent sidebar provides predictable navigation and allows users to switch between modules without losing context.

**Impact**

Improves discoverability and consistency throughout the application.

---

### Decision 3 — Global Search Accessible Everywhere

**Context**

Users frequently need to locate resources without navigating through multiple modules.

**Rationale**

Providing a global search allows users to quickly locate issues, projects, cycles, and members from anywhere in the application.

**Impact**

Reduces navigation time and improves productivity.

---

### Decision 4 — Workspace Switching Uses a Modal

**Context**

Changing workspaces is an occasional context switch rather than a primary workflow.

**Rationale**

A modal allows users to switch workspaces without leaving their current screen or disrupting the overall navigation structure. Workspace names are non-unique display labels, so matching names are distinguished with icon and role or ownership context while selection uses the immutable internal Workspace identifier.

**Impact**

Provides a lightweight workspace switching experience without treating a user-facing name as identity.

---

## 8.3 Information Architecture Decisions

### Decision 5 — Projects, Cycles, and Issues are Independent Entities

**Context**

Each entity represents a distinct level of work organization.

**Rationale**

Treating them as independent modules improves scalability and allows users to navigate directly to each resource. Projects and Cycles do not directly contain or own one another; Issues independently connect to either entity.

**Impact**

Supports future product expansion without restructuring the application. Any Project–Cycle connection shown in the interface is derived from Issues associated with both.

---

### Decision 6 — Comments Belong to Issues

**Context**

Discussions are always related to a specific issue.

**Rationale**

Embedding comments within Issue Details keeps conversations contextual and avoids creating a disconnected communication system.

**Impact**

Improves collaboration while maintaining focus on the associated work item.

---

### Decision 7 — Notifications are Globally Accessible

**Context**

MVP notifications originate from Issue assignment or reassignment and mentions in Issue comments.

**Rationale**

A centralized notification panel provides consistent access to these focused events regardless of the user's current location.

**Impact**

Users remain informed without interrupting their workflow.

---

## 8.4 Interaction Decisions

### Decision 8 — Creation Uses Modals

**Context**

Creating resources should be fast and minimally disruptive.

**Rationale**

Using modals allows users to create new entities while remaining within their current context. In an active Workspace, the Global Create Menu exposes Issue, Project, and Cycle creation according to the current user's permissions and omits unauthorized actions.

**Impact**

Reduces unnecessary page transitions, improves workflow efficiency, and keeps the global action surface aligned with RBAC.

---

### Decision 9 — Primary Entities Use Dedicated Detail Pages

**Context**

Projects, Cycles, and Issues contain significant information and multiple management actions.

**Rationale**

Dedicated pages provide sufficient space for complex information and future feature expansion.

**Impact**

Creates scalable interfaces without overwhelming users.

---

### Decision 10 — Destructive Actions Require Confirmation

**Context**

Deleting or archiving resources and transferring Workspace or Project ownership may have significant consequences.

**Rationale**

Confirmation dialogs help prevent accidental destructive actions and unintended Workspace ownership transfers. Project ownership is lower risk because it grants no permissions, so a Workspace Owner or Admin can change it directly from a non-archived Project after selecting a current workspace member. For Project deletion, the dialog states that deletion is permanent, shows the number of associated Issues, and explains that those Issues will be unassigned rather than deleted. A Workspace must be archived before deletion, and its Owner must type the exact workspace name before permanent deletion can proceed.

**Impact**

Improves user confidence and reduces irreversible mistakes. Project deletion preserves its Issues, while the stronger Workspace deletion gate protects the complete workspace data boundary.

---

### Decision 11 — Workflow Status Updates are Inline

**Context**

Changing Issue or Project status is a frequent action in the application.

**Rationale**

Inline controls reduce interaction cost compared to opening additional dialogs or pages. Issue workflow status follows its defined transitions, while a Project may switch freely between Planned, Active, and Completed. Archived is excluded from inline Project status controls and remains a separate confirmed action.

**Impact**

Enables faster Issue and Project management without allowing inline status changes to bypass Archive or Restore confirmation.

---

### Decision 12 — Editing Remains Contextual

**Context**

Most edits involve modifying existing information rather than navigating elsewhere.

**Rationale**

Contextual editing through modals or inline controls minimizes disruption.

**Impact**

Maintains user focus while reducing navigation.

---

## 8.5 Workflow Decisions

### Decision 13 — Simple Four-State Issue Workflow

**Context**

The MVP focuses on a lightweight project management experience.

**Rationale**

A fixed workflow of **Backlog → Todo → In Progress → Done** balances simplicity with effective work tracking. Blocked is an independent flag for unfinished issues, not a fifth workflow state, and does not introduce issue dependencies.

**Impact**

Reduces complexity while supporting common development workflows and makes impeded work visible without complicating status transitions.

---

### Decision 14 — Archived Resources Become Read-Only

**Context**

Historical information should remain accessible without allowing accidental modification.

**Rationale**

Read-only archived resources preserve historical accuracy while preventing unintended changes. Authorized users can restore an archived resource when it needs to return to active use.

**Impact**

Maintains data integrity while keeping archival reversible. Restoration preserves the resource's data and history and returns it to its permitted pre-archive state.

---

### Decision 15 — Cycle Lifecycle Uses Explicit Non-Overlapping Transitions

**Context**

Cycles define the workspace delivery schedule, so overlapping dates or arbitrary status changes would make the current iteration ambiguous.

**Rationale**

Non-archived Cycle date ranges never overlap. Lifecycle changes use explicit Start, Complete, Reopen, Archive, and Restore actions instead of a generic status selector. Only one Cycle may be Active, and an Active Cycle must be completed before it can be archived.

**Impact**

Creation, date editing, restoration, and reopening include conflict checks. The interface exposes only actions valid for the Cycle's current state and explains how to resolve blocked transitions.

---

### Decision 16 — Dashboard Serves as the Productivity Hub

**Context**

Users frequently need quick access to current work.

**Rationale**

Centralizing assigned work, projects, cycles, and recent activity encourages users to begin work directly from the Dashboard. Quick actions follow workspace permissions: all roles can create Issues, search, and view assigned work, while only Owners and Admins see Project and Cycle creation.

**Impact**

Improves efficiency and reduces unnecessary navigation.

---

## 8.6 Consistency Decisions

### Decision 17 — Shared Interaction Patterns Across Modules

**Context**

Projects, Cycles, and Issues perform similar management operations.

**Rationale**

Using consistent patterns for creating, viewing, editing, archiving, and deleting resources reduces the learning curve.

**Impact**

Provides a predictable user experience throughout the application.

---

### Decision 18 — Consistent Detail Page Structure

**Context**

Primary entities expose multiple sections and management actions.

**Rationale**

Using a consistent layout for entity detail pages helps users transfer knowledge between modules.

**Impact**

Improves usability and simplifies future feature additions.

---

### Decision 19 — Empty States Always Guide Users Forward

**Context**

Users should never encounter blank interfaces without guidance.

**Rationale**

Every empty state includes a meaningful explanation and, where appropriate, a clear next action. Actions are filtered by workspace permissions, and unauthorized actions are omitted instead of shown disabled.

**Impact**

Improves discoverability and encourages continued engagement.

---

## 8.7 Future Scalability Decisions

### Decision 20 — Post-MVP Features are Intentionally Deferred

**Context**

The MVP prioritizes delivering a focused and maintainable product.

**Rationale**

Features such as Kanban, Timeline, Calendar, Attachments, Rich Text Editing, Advanced Search, and Shared Saved Views are excluded from the initial release to reduce complexity.

**Impact**

Allows the MVP to remain focused while providing a clear path for future expansion.

---

### Decision 21 — Architecture Supports Incremental Growth

**Context**

Shipyard is expected to evolve beyond the MVP.

**Rationale**

The information architecture, navigation, and interaction patterns are designed so that future modules and capabilities can be added without requiring significant redesign.

**Impact**

Supports long-term maintainability and product evolution.
