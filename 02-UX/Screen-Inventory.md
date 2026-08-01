# 5. Screen Inventory

## 5.1 Overview

This section identifies every screen, modal, drawer, dialog, and overlay required to implement the Shipyard MVP. The inventory is derived from the Product Requirements Document (PRD), Information Architecture, Navigation, and User Flows, ensuring that every user interaction has a corresponding UI surface.

---

## 5.2 Authentication

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Login | Page | Application entry, Logout redirect | Authenticate existing users and grant access to the workspace. |
| Registration | Page | Login page | Allow new users to create an account. |
| Forgot Password | Page | Login page | Allow users to request a password reset. |
| Reset Password | Page | Password reset link | Allow users to create a new password after identity verification. |

---
## 5.3 Workspace

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Workspace Onboarding | Page | First login, users without a workspace | Guide first-time users through creating their initial workspace. |
| Create Workspace | Modal | Workspace Onboarding, Workspace Switcher | Allow users to create a new workspace. |
| Workspace Switcher | Modal | Sidebar, Header, User Menu | Display available workspaces with name, icon, and role or ownership context, allowing duplicate names to remain distinguishable and selectable. |
| Join Workspace Invitation | Page | Invitation link | Allow invited users to review and accept a workspace invitation. |
| Workspace Loading | Full-Screen State | Login, Workspace Switcher | Display the workspace loading state while preparing the selected workspace. |
| Archive Workspace Confirmation | Dialog | Workspace Settings → Danger Zone | Confirm making an active workspace read-only before it can be restored or permanently deleted. |
| Archived Workspace Summary | Page | Workspace Switcher → Archived Workspaces | Display an archived workspace in read-only mode and allow its Owner to restore or permanently delete it. |
| Restore Workspace Confirmation | Dialog | Archived Workspace Summary | Confirm restoring an archived workspace to active use. |
| Delete Workspace Confirmation | Dialog | Archived Workspace Summary | Identify the selected workspace with name, icon, and ownership context, then require its exact name before permanently deleting all workspace-scoped data and memberships. |

---

## 5.4 Members

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Members Directory | Page | Sidebar → Members | Display all workspace members, their roles, and their current status to every workspace member. |
| Invite Members | Modal | Members Directory | Allow Owners to invite Members or Admins and allow Admins to invite Members. |
| Member Details | Drawer | Members Directory | Display member information and role-appropriate management actions. |
| Change Role Confirmation | Dialog | Member Details | Allow an Owner to confirm a Member or Admin role change. |
| Remove Member Confirmation | Dialog | Member Details | Confirm removal when an Owner removes a Member or Admin, or an Admin removes a Member; show any owned Projects that will transfer automatically to the Workspace Owner. |
| Transfer Workspace Ownership Confirmation | Dialog | Member Details, Workspace Settings | Confirm transferring Workspace ownership to an existing Member or Admin and changing the current Owner to Admin. |
| Leave Workspace Confirmation | Dialog | User Menu, Workspace Settings | Confirm that a Member or Admin wants to leave, show any owned Projects that will transfer automatically to the Workspace Owner, and require the current Owner to transfer Workspace ownership first. |

---
## 5.5 Projects

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Projects List | Page | Sidebar → Projects, Dashboard | Display all projects within the active workspace. |
| Project Details | Page | Projects List, Dashboard, Search, Notifications | Display project information, Project Owner, progress, associated issues, project activity, and any cycle information derived from those issues. |
| Create Project | Modal | Projects List, Global Create Menu | Allow authorized users to create a Project with a name unique within the workspace. |
| Edit Project | Modal | Project Details | Allow authorized users to update project information while preserving workspace-scoped name uniqueness. |
| Change Project Owner | Modal | Non-archived Project Details | Allow Workspace Owners and Admins to transfer Project ownership to any other current workspace member without changing that member's role or permissions. |
| Update Project Status | Dropdown | Non-archived Project Details, Projects List | Allow editors to switch freely between Planned, Active, and Completed; Archived is excluded. |
| Archive Project Confirmation | Dialog | Non-archived Project Details | Confirm archiving a project before it becomes read-only. |
| Restore Project Confirmation | Dialog | Archived Projects, Project Details | Confirm returning an archived project to its stored operational status. |
| Delete Project Confirmation | Dialog | Project Details | Confirm permanent deletion, show the number of affected issues, and explain that those issues will be unassigned rather than deleted. |

---
## 5.6 Cycles

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Cycles List | Page | Sidebar → Cycles, Dashboard | Display all cycles within the active workspace. |
| Cycle Details | Page | Cycles List, Dashboard, Search, Notifications | Display cycle information, progress, associated issues, statistics, and activity. |
| Create Cycle | Modal | Cycles List, Global Create Menu | Allow authorized users to create a Cycle with a name unique within the workspace and a non-overlapping date range. |
| Edit Cycle | Modal | Planned or Active Cycle Details | Allow authorized users to update Cycle information while preserving workspace-scoped name uniqueness and non-overlapping dates. |
| Start Cycle Confirmation | Dialog | Planned Cycle Details | Confirm starting a Planned cycle when no other cycle is Active. |
| Complete Cycle Confirmation | Dialog | Cycle Details | Confirm completion of a cycle and finalize the iteration. |
| Reopen Cycle Confirmation | Dialog | Cycle Details | Confirm reopening a completed cycle. |
| Archive Cycle Confirmation | Dialog | Planned or Completed Cycle Details | Confirm archiving a non-active cycle. |
| Restore Cycle Confirmation | Dialog | Archived Cycles, Cycle Details | Confirm restoring an archived cycle. |
| Delete Cycle Confirmation | Dialog | Future Planned Cycle Details | Confirm permanent deletion and explain that associated issues will be unassigned. |

----
## 5.7 Issues

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Issues List | Page | Sidebar → Issues, Dashboard | Display all issues within the active workspace. |
| Issue Details | Page | Issues List, Dashboard, Search, Notifications, Project Details, Cycle Details | Display issue information, workflow status, planning details, activity history, and management actions. |
| Create Issue | Modal | Issues List, Project Details, Cycle Details, Dashboard, Global Create Menu | Allow users to create a new issue with an optional Priority that defaults to No Priority. |
| Edit Issue | Modal | Issue Details | Allow authorized users to update issue information. |
| Assign Issue | Modal | Issue Details | Allow users to assign or reassign an issue. |
| Update Workflow Status | Dropdown | Issue Details, Issues List | Allow users to move an issue through the workflow (Backlog → Todo → In Progress → Done). |
| Update Blocked State | Inline Control | Issue Details, Issues List | Allow users to mark an unfinished issue as blocked with an optional reason or clear its blocked state. |
| Archive Issue Confirmation | Dialog | Issue Details | Confirm archiving an issue. |
| Restore Issue Confirmation | Dialog | Archived Issues, Issue Details | Confirm restoring an archived issue. |
| Delete Issue Confirmation | Dialog | Issue Details | Confirm permanent deletion of an issue. |

---
## 5.8 Collaboration

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Comments Section | Section | Issue Details | Display issue discussions, mentions, and the complete comment history. |
| Edit Comment | Inline Editor | Comments Section | Allow users to modify their own comments without leaving the issue. |
| Delete Comment Confirmation | Dialog | Comments Section | Confirm deletion of a comment before removing it permanently. |
| Mention Suggestions | Popover | Comment Editor | Display matching workspace members while typing '@' in a comment. |
| Notifications Panel | Panel | Header Navigation | Display recent notifications and provide quick access to related resources. |
| Clear All Notifications Confirmation | Dialog | Notifications Panel | Confirm removal of all notifications. |

---
## 5.9 Dashboard

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Dashboard | Page | Login, Workspace Switcher, Sidebar | Display a personalized overview of the user's workspace, assigned work, projects, cycles, and recent activity. |
| Recent Activity Feed | Section | Dashboard | Display the latest workspace activity and provide quick navigation to related resources. |
| Quick Actions | Section | Dashboard | Provide shortcuts to common actions such as creating an issue, searching the workspace, or viewing assigned work. |

---
## 5.10 Global Actions

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Global Create Menu | Popover | Global Create Button | Display Create Issue to every workspace role and Create Project or Create Cycle only to Owners and Admins; omit unauthorized actions. |

---
## 5.11 Search

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Global Search | Modal | Header Navigation, Keyboard Shortcut | Allow users to quickly search for issues, projects, cycles, and members across the active workspace. |
| Search Results | Section | Global Search | Display matching resources and allow users to navigate directly to them. |
| Filter Panel | Drawer | Issues List, Projects List, Cycles List | Allow users to refine displayed resources using available filters. |
| Save View | Modal | Filter Panel | Allow users to save the current filter configuration as a private view. |
| Saved Views | Dropdown | Resource List Toolbar | Display previously saved views and allow users to apply, rename, delete, or set a default view. |

---
## 5.12 Settings

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Profile Settings | Page | User Menu → Account Settings | Allow users to manage their display name, profile picture, and account email. |
| Security Settings | Page | Account Settings | Allow users to change their password. |
| Appearance Settings | Page | Account Settings | Allow users to select the Light, Dark, or System theme. |
| Change Email | Modal | Profile Settings | Allow users to update their account email after providing their current password. |
| Workspace Settings | Page | Sidebar → Workspace Settings | Allow authorized users to manage workspace configuration. |
| Change Password | Modal | Security Settings | Allow users to securely update their account password. |
