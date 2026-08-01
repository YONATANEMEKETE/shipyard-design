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
| Logout Confirmation *(Optional)* | Dialog | User account menu | Confirm logout before ending the current session, if confirmation is enabled. |

---
## 5.3 Workspace

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Workspace Onboarding | Page | First login, users without a workspace | Guide first-time users through creating their initial workspace. |
| Create Workspace | Modal | Workspace Onboarding, Workspace Switcher | Allow users to create a new workspace. |
| Workspace Switcher | Modal | Sidebar, Header, User Menu | Display available workspaces and allow users to switch between them. |
| Join Workspace Invitation | Page | Invitation link | Allow invited users to review and accept a workspace invitation. |
| Workspace Loading | Full-Screen State | Login, Workspace Switcher | Display the workspace loading state while preparing the selected workspace. |

---

## 5.4 Members

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Members Directory | Page | Sidebar → Members | Display all workspace members, their roles, and their current status. |
| Invite Members | Modal | Members Directory | Invite new users to join the workspace via email. |
| Member Details | Drawer | Members Directory | Display member information and available management actions. |
| Change Role Confirmation | Dialog | Member Details | Confirm role changes before updating member permissions. |
| Remove Member Confirmation | Dialog | Member Details | Confirm removal of a member from the workspace. |

---
## 5.5 Projects

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Projects List | Page | Sidebar → Projects, Dashboard | Display all projects within the active workspace. |
| Project Details | Page | Projects List, Dashboard, Search, Notifications | Display project information, progress, associated issues, project activity, and any cycle information derived from those issues. |
| Create Project | Modal | Projects List | Allow users to create a new project. |
| Edit Project | Modal | Project Details | Allow authorized users to update project information. |
| Archive Project Confirmation | Dialog | Project Details | Confirm archiving a project before it becomes read-only. |
| Restore Project Confirmation | Dialog | Project Details | Confirm restoring an archived project. |
| Delete Project Confirmation | Dialog | Project Details | Confirm permanent deletion of a project. |

---
## 5.6 Cycles

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Cycles List | Page | Sidebar → Cycles, Dashboard | Display all cycles within the active workspace. |
| Cycle Details | Page | Cycles List, Dashboard, Search, Notifications | Display cycle information, progress, associated issues, statistics, and activity. |
| Create Cycle | Modal | Cycles List, Global Create Menu | Allow authorized users to create a workspace cycle. |
| Edit Cycle | Modal | Cycle Details | Allow authorized users to update cycle information. |
| Complete Cycle Confirmation | Dialog | Cycle Details | Confirm completion of a cycle and finalize the iteration. |
| Reopen Cycle Confirmation | Dialog | Cycle Details | Confirm reopening a completed cycle. |
| Archive Cycle Confirmation | Dialog | Cycle Details | Confirm archiving a cycle. |

----
## 5.7 Issues

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Issues List | Page | Sidebar → Issues, Dashboard | Display all issues within the active workspace. |
| Issue Details | Page | Issues List, Dashboard, Search, Notifications, Project Details, Cycle Details | Display issue information, workflow status, planning details, activity history, and management actions. |
| Create Issue | Modal | Issues List, Project Details, Cycle Details, Dashboard | Allow users to create a new issue. |
| Edit Issue | Modal | Issue Details | Allow authorized users to update issue information. |
| Assign Issue | Modal | Issue Details | Allow users to assign or reassign an issue. |
| Update Workflow Status | Dropdown | Issue Details, Issues List | Allow users to move an issue through the workflow (Backlog → Todo → In Progress → Done). |
| Archive Issue Confirmation | Dialog | Issue Details | Confirm archiving an issue. |
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
## 5.10 Search

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Global Search | Modal | Header Navigation, Keyboard Shortcut | Allow users to quickly search for issues, projects, cycles, and members across the active workspace. |
| Search Results | Section | Global Search | Display matching resources and allow users to navigate directly to them. |
| Filter Panel | Drawer | Issues List, Projects List, Cycles List | Allow users to refine displayed resources using available filters. |
| Save View | Modal | Filter Panel | Allow users to save the current filter configuration as a private view. |
| Saved Views | Dropdown | Resource List Toolbar | Display previously saved views and allow users to apply, rename, delete, or set a default view. |

---
## 5.11 Settings

| Screen | Type | Entry Points | Purpose |
|---------|------|--------------|----------|
| Profile Settings | Page | User Menu → Settings | Allow users to manage their personal profile information. |
| Security Settings | Page | Profile Settings | Allow users to change their password and manage account security. |
| Workspace Settings | Page | Sidebar, User Menu | Allow authorized users to manage workspace configuration. |
| Change Password | Modal | Security Settings | Allow users to securely update their account password. |
| Logout Confirmation *(Optional)* | Dialog | User Menu | Confirm logout before ending the current session, if confirmation is enabled. |
