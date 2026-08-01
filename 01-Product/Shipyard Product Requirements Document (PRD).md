# Shipyard Product Requirements Document (PRD)

- **Version:** 1.0
- **Status:** Draft - Approved for Product Backlog Creation
- **Product:** Shipyard
- **Document Owner:** Yonatane Mekete
- **Last Updated:** July 2026

## 1. Introduction

### Purpose

This Product Requirements Document (PRD) defines the functional and non-functional requirements for Shipyard, a modern open-source project management platform built specifically for small software engineering teams.

The PRD serves as the primary reference for product management, design, frontend development, backend development, quality assurance, and future contributors. It translates the product vision into clear, implementable requirements that guide the development of the MVP and future releases.

### Product Summary

Shipyard enables small engineering teams to plan, organize, collaborate on, and deliver software through a fast, opinionated, and developer-first workflow.

The product centers around five core workflows:

- Managing work
- Planning projects
- Running development cycles
- Collaborating with teammates
- Tracking progress

Rather than maximizing customization, Shipyard emphasizes simplicity, consistency, and excellent defaults, allowing teams to focus on building software instead of configuring their project management tool.

### Target Audience

Shipyard is designed primarily for:

- Startup engineering teams (2-30 people)
- Indie hackers
- Solo developers
- Small software agencies

It is not intended to compete with enterprise-focused project management platforms or support highly customized organizational workflows.

### Product Principles

Every requirement in this document should align with these principles:

- Simplicity over complexity
- Build for software teams first
- Opinionated by default
- Speed first
- Consistency across workflows
- Quality over quantity

When requirements conflict, these principles should guide decision-making.

### MVP Definition

Version 1 is considered complete when a small engineering team can:

- Create a workspace
- Invite team members
- Create and organize projects
- Manage issues
- Plan and complete development cycles
- Collaborate through comments
- Track team progress
- Use Shipyard as their primary project management tool

### Document Scope

This PRD defines:

- Functional requirements
- User stories
- Acceptance criteria
- Business rules
- Permissions
- System behaviors
- MVP boundaries

It does not define:

- UI design specifications
- Database schema
- API contracts
- Technical architecture
- Implementation details

## 2. Product Goals

### Purpose

Product goals describe the outcomes Shipyard is designed to achieve. Every feature included in the MVP should contribute directly to one or more of these goals.

### Goal 1 - Simplify Software Project Management

Enable small software teams to organize and manage their work without the complexity of enterprise project management tools.

#### Success Indicators

- Teams can start using Shipyard with minimal onboarding.
- Common workflows require little to no configuration.
- Users spend more time building software than managing the tool.

### Goal 2 - Help Teams Ship Software Faster

Reduce the friction involved in planning, assigning, tracking, and completing development work.

#### Success Indicators

- Teams can easily prioritize work.
- Development cycles remain organized.
- Progress is always visible.
- Work moves smoothly from backlog to completion.

### Goal 3 - Deliver an Exceptional Developer Experience

Provide a fast, intuitive, and consistent experience designed specifically for software engineering teams.

#### Success Indicators

- Fast page loads and interactions.
- Keyboard-friendly workflows.
- Consistent UI patterns.
- Minimal clicks for common actions.

### Goal 4 - Build a Production-Grade Open Source Platform

Develop Shipyard with production-quality engineering practices while keeping it accessible for contributors and self-hosting.

#### Success Indicators

- Clear documentation.
- Easy local development setup.
- Well-structured architecture.
- Maintainable codebase.
- Transparent contribution process.

### Goal 5 - Establish a Reusable Foundation

Create architecture, patterns, and documentation that can be reused across future Build Stage products.

This is an internal engineering goal, but it's important because Shipyard is the first project in the Build Stage ecosystem.

#### Success Indicators

- Reusable authentication patterns.
- Shared UI components.
- Scalable project structure.
- Consistent engineering standards.

### Goal Prioritization

Not all goals have the same priority. If trade-offs arise, we'll prioritize them in this order:

| Priority | Goal |
| --- | --- |
| P1 | Simplify software project management |
| P1 | Help teams ship software faster |
| P2 | Deliver an exceptional developer experience |
| P2 | Build a production-grade open-source platform |
| P3 | Establish a reusable foundation |

This ordering helps resolve conflicts. For example, if a feature improves developer experience but significantly complicates the product for users, we'd choose the simpler product because our highest priorities are serving small software teams.

### Product Success Statement

When Shipyard reaches MVP, we want a small engineering team to be able to say:

> "We replaced our project management tool with Shipyard and never felt like we were missing the essentials."

That sentence captures the outcome we're aiming for better than any individual metric.

## 3. Glossary

| Term | Definition |
| --- | --- |
| Workspace | The top-level container for a team and its data. |
| Member | A user who belongs to a workspace. |
| Issue | A unit of work such as a bug, task, or feature. |
| Project | A collection of related issues with a shared goal. |
| Cycle | A fixed development period used to organize work. |
| Assignee | The member responsible for an issue. |
| Label | A tag used to categorize issues. |
| Blocked | An issue flag indicating that work cannot currently proceed; it is separate from workflow status. |
| Notification | An in-app alert about relevant activity. |

## 4. User Personas

### Persona 1 - Startup Founder / Product Manager

#### Responsibilities

- Plan product roadmap
- Prioritize work
- Create projects and issues
- Track team progress

#### Goals

- Keep development organized
- Ensure the team ships on time
- Maintain visibility across projects

#### Pain Points

- Too much time spent managing tools
- Difficult to see overall progress
- Complex project management software

#### Primary Actions in Shipyard

- Create projects
- Create and prioritize issues
- Plan cycles
- Review dashboards
- Monitor team progress

### Persona 2 - Software Engineer

#### Responsibilities

- Implement features
- Fix bugs
- Update issue status
- Collaborate with teammates

#### Goals

- Know what to work on
- Track personal tasks
- Communicate efficiently

#### Pain Points

- Unclear priorities
- Losing context across discussions
- Switching between multiple tools

#### Primary Actions in Shipyard

- View assigned issues
- Update issue status
- Comment on issues
- Complete tasks
- Receive notifications

### Persona 3 - Engineering Lead

#### Responsibilities

- Coordinate developers
- Plan development cycles
- Remove blockers
- Monitor delivery

#### Goals

- Balance workload
- Keep projects on schedule
- Identify bottlenecks early

#### Pain Points

- Limited visibility into team progress
- Difficulty tracking blocked work
- Too much manual coordination

#### Primary Actions in Shipyard

- Manage cycles
- Assign issues
- Track project progress
- Review dashboards
- Resolve blockers

### Why Only These Three?

These personas represent the majority of users in a small software team:

- **Founder / Product Manager:** Decides what gets built.
- **Engineering Lead:** Decides who builds it and when.
- **Software Engineer:** Builds it.

Designing for these three covers nearly every workflow in our MVP.

## 5. Functional Requirements

### Standard Feature Template

Every feature in this section follows the same structure:

- Overview
- Goals
- User Stories
- Functional Requirements
- Business Rules
- Acceptance Criteria
- Edge Cases
- Future Enhancements (optional)

### 5.1 Authentication

#### Overview

Authentication allows users to securely create an account, sign in, access their workspaces, and manage their sessions.

Authentication is the entry point to Shipyard and must provide a secure, frictionless experience while supporting future workspace collaboration.

#### Goals

- Secure user accounts.
- Fast onboarding.
- Support individual users and teams.
- Minimize login friction.

#### User Stories

- As a new user, I want to create an account so I can start using Shipyard.
- As a returning user, I want to sign in quickly.
- As a logged-in user, I want to remain signed in across sessions.
- As a user, I want to sign out securely.

#### Functional Requirements

##### Registration

- Users can register using an email and password.
- Email must be unique.
- Password must meet minimum security requirements.
- New users are automatically signed in after successful registration.

##### Login

- Users can log in with their email and password.
- Invalid credentials display a generic error message.
- Successful login redirects users to their workspace selection or dashboard.

##### Logout

- Users can log out from any device.
- Selecting Logout immediately invalidates the current session without a confirmation step.
- Successful logout redirects the user to the login page.

##### Session Management

- Sessions persist across browser refreshes.
- Expired sessions redirect users to the login page.
- Protected pages require authentication.

##### Password Management

- Users can request a password reset.
- Password reset links expire after a configurable time.
- Users can change their password after logging in.

#### Business Rules

- Email addresses are unique.
- Passwords are never stored in plain text.
- Authentication is required for all protected routes.
- Users may belong to multiple workspaces.
- Authentication is separate from workspace permissions.

#### Acceptance Criteria

##### Registration

Given a new user, when valid registration details are submitted, then the account is created and the user is signed in.

##### Login

Given an existing account, when valid credentials are entered, then access is granted.

##### Logout

Given an authenticated user, when they select Logout, then the current session is invalidated immediately without confirmation and they are redirected to the login page.

##### Protected Routes

Given an unauthenticated visitor, when they access a protected page, then they are redirected to the login screen.

##### Password Reset

Given a registered email, when a password reset is requested, then a reset email is sent.

#### Edge Cases

- Duplicate email registration.
- Invalid email format.
- Weak password.
- Expired password reset link.
- Invalid reset token.
- Expired session.
- Attempt to access protected pages without authentication.

#### Future Enhancements

- OAuth (Google, GitHub).
- Passkeys.
- Two-factor authentication.
- Single Sign-On (SSO).
- Device/session management.

### 5.2 Workspace

#### Overview

A workspace is the primary container for a team's data. It groups members, projects, issues, cycles, and settings into an isolated environment where a team collaborates.

#### Goals

- Provide an isolated environment for each team.
- Allow users to create and join workspaces.
- Keep workspace data separated.
- Support collaboration within a team.

#### User Stories

- As a new user, I want to create a workspace so I can start managing my team's work.
- As a user, I want to join an existing workspace through an invitation.
- As a member, I want to switch between my workspaces.
- As a workspace owner, I want to manage workspace information.

#### Functional Requirements

##### Workspace Creation

- Users can create a new workspace.
- A workspace must have a name.
- Workspace names are display labels and do not need to be unique.
- A workspace may have an icon or logo.
- The system assigns every workspace an immutable, unique internal identifier.
- The creator becomes the workspace owner.
- A newly created workspace is immediately available for use.

##### Workspace Management

- Workspace owners can update the workspace name.
- Workspace owners can update the workspace icon.
- Workspace owners can archive the workspace.
- Archived workspaces cannot be actively used.
- Workspace owners can restore an archived workspace.
- Restoring a workspace makes it active and usable again.
- Workspace owners can permanently delete an Archived workspace.
- The Owner must enter the exact workspace name before permanent deletion can be confirmed.

##### Workspace Membership

- Users can belong to multiple workspaces.
- Each workspace maintains its own member list.
- Members only have access to workspaces they belong to.

##### Workspace Switching

- Users can view all workspaces they belong to.
- Users can switch between workspaces.
- The Workspace Switcher displays each workspace's name, icon, and the user's role to help distinguish duplicate names.
- Switching workspaces updates all visible data to the selected workspace.

#### Business Rules

- Every project belongs to one workspace.
- Every issue belongs to one workspace.
- Every cycle belongs to one workspace.
- Duplicate workspace names are allowed, including among workspaces visible to the same user.
- Routing, permissions, invitations, and data relationships use the immutable workspace identifier, never the workspace name.
- Renaming a workspace does not change its internal identifier or break existing references.
- Workspace data is isolated from other workspaces.
- An active workspace must be archived before it can be permanently deleted.
- Deleting an Archived workspace removes all workspace-scoped resources, settings, history, memberships, and invitations.
- Workspace deletion does not delete member user accounts or their data in other workspaces.
- Workspace deletion is irreversible and must succeed or fail as one operation; a failed deletion leaves the Archived workspace unchanged.
- Only workspace owners can archive, restore, or delete a workspace.

#### Acceptance Criteria

##### Create Workspace

Given an authenticated user, when they create a workspace with valid information, then the workspace receives a unique internal identifier and is created even when another workspace has the same display name, and the user becomes its owner.

##### Join Workspace

Given a valid invitation, when a user accepts it, then they become a member of the workspace.

##### Switch Workspace

Given a user belongs to multiple workspaces, when they select a different workspace, then all displayed data updates to the selected workspace.

##### Archive Workspace

Given a workspace owner, when they confirm Archive, then the workspace becomes read-only and is no longer active.

##### Restore Workspace

Given a workspace owner and an archived workspace, when they restore it, then the workspace becomes active and usable again with its data preserved.

##### Delete Workspace

Given a workspace owner and an Archived workspace, when they enter the exact workspace name and confirm deletion, then the workspace and all workspace-scoped data and memberships are permanently removed while member user accounts remain intact.

#### Edge Cases

- Multiple accessible workspaces share the same display name.
- User attempts to access a workspace they are not a member of.
- The Owner attempts to leave before transferring ownership.
- Owner attempts to delete an active workspace before archiving it.
- Entered deletion confirmation does not exactly match the workspace name.
- Permanent deletion fails before all workspace-scoped data can be removed.
- Invalid or expired invitation link.
- Archived workspace accessed directly via URL.

#### Future Enhancements

- Workspace templates.
- Workspace duplication.
- Custom branding.
- Organization-level management.

### 5.3 Members

#### Overview

The Members feature allows teams to collaborate by inviting users to a workspace, assigning roles, and managing membership.

#### Goals

- Allow teams to grow through invitations.
- Control who has access to a workspace.
- Manage member roles and permissions.
- Keep workspace membership secure and organized.

#### User Stories

- As a workspace owner, I want to invite teammates.
- As a user, I want to accept an invitation and join a workspace.
- As an owner, I want to change a member's role.
- As an owner, I want to remove members from my workspace.
- As a member, I want to leave a workspace.

#### Functional Requirements

##### Invitations

- Workspace Owners and Admins can invite users.
- Owners can invite users as Members or Admins.
- Admins can invite users only as Members.
- The Owner role cannot be assigned through an invitation.
- Invitations are sent using an email address.
- Users can accept or decline an invitation.
- Invitations expire after a defined period.
- Invitations can be revoked before they are accepted.
- Users cannot accept the same invitation more than once.

##### Member Management

- All workspace members can view the member directory.
- Member information includes name, email, role, and join date.
- Owners can remove Members and Admins.
- Admins can remove Members but cannot remove Admins or Owners.
- Only Owners can change member roles.
- Members can leave a workspace voluntarily.
- The Owner can transfer workspace ownership to an existing Member or Admin.
- Ownership transfer atomically promotes the recipient to Owner and changes the transferring Owner to Admin.
- If a removed or departing Member or Admin owns Projects, those Projects transfer automatically to the Workspace Owner as part of the membership change.

##### Roles

The MVP supports three roles:

###### Owner

- Full workspace access.
- Manage members.
- Manage workspace settings.
- Archive the active workspace or restore or delete it after archival.

###### Admin

- Manage projects.
- Manage issues.
- Invite users as Members.
- Remove Members.
- Manage cycles.

###### Member

- View workspace content.
- Create and update issues.
- Comment on issues.
- Participate in projects and cycles.

#### Business Rules

- Every member belongs to a workspace.
- A user may belong to multiple workspaces.
- Every workspace has exactly one Owner.
- Removed members immediately lose access.
- Pending invitations do not grant access.
- Users cannot invite themselves.
- Users already in the workspace cannot be invited again.
- Admins cannot invite users as Admins or Owners.
- Admins cannot remove Admins or Owners.
- Only Owners can change Member/Admin roles or transfer Workspace ownership.
- Automatic Project reassignment during member removal or departure always targets the Workspace Owner; an Admin performing a permitted Member removal cannot choose another recipient.

#### Acceptance Criteria

##### Invite Member

Given a user with permission, when they invite another user, then a pending invitation is created.

##### Accept Invitation

Given a valid invitation, when it is accepted, then the user becomes a workspace member.

##### Remove Member

Given an Owner or Admin with permission to remove the selected Member, when they confirm removal, then any Projects owned by that Member transfer to the Workspace Owner and the Member immediately loses workspace access.

##### Leave Workspace

Given a Member or Admin, when they leave a workspace, then any Projects they own transfer to the Workspace Owner and they are removed. The Owner must transfer Workspace ownership and become an Admin before leaving.

##### Transfer Workspace Ownership

Given the current Owner selects an existing Member or Admin and confirms ownership transfer, then the recipient becomes the new Owner and the transferring Owner becomes an Admin in one atomic operation.

#### Edge Cases

- Invitation sent to an existing member.
- Invitation expires before acceptance.
- Invitation revoked before acceptance.
- User tries to use an expired invitation.
- Owner attempts to remove themselves before transferring ownership.
- Owner attempts to leave before transferring ownership.
- Ownership transfer target is no longer a workspace member.
- Ownership transfer is interrupted before both role changes complete.
- Removed member tries to access the workspace using an old link.

#### Future Enhancements

- Custom roles.
- Permission editor.
- Team groups.
- Bulk member invitations.
- Organization-wide member management.

### 5.4 Dashboard

#### Overview

The Dashboard is the default landing page within a workspace. It provides users with a personalized overview of their work, team activity, and project progress.

#### Goals

- Give users immediate visibility into their work.
- Surface important updates.
- Reduce time spent navigating the application.
- Help teams stay aligned.

#### User Stories

- As a member, I want to see my assigned work.
- As a team lead, I want to monitor team progress.
- As a user, I want to quickly identify overdue or blocked work.
- As a user, I want to resume work without searching for it.

#### Functional Requirements

##### Personal Overview

- Display issues assigned to the current user.
- Display issues created by the current user.
- Display recently viewed issues.
- Highlight overdue issues.
- Highlight blocked issues assigned to the current user.

##### Workspace Overview

- Display active projects.
- Display the current active cycle.
- Display recent workspace activity.
- Display recently completed issues.

##### Quick Actions

- All roles can create a new Issue.
- All roles can search the workspace.
- All roles can view their assigned Issues.
- Owners and Admins can create a new Project.
- Owners and Admins can create a new Cycle.

##### Activity Feed

- Show recent issue updates.
- Show newly created issues.
- Show completed issues.
- Show member comments.
- Show project updates.

#### Business Rules

- Dashboard content is limited to the selected workspace.
- Users only see information they have permission to access.
- Personal sections are specific to the logged-in user.
- Activity is displayed in reverse chronological order.
- Archived items are excluded unless explicitly requested.
- Dashboard quick actions and empty-state actions are filtered by the current user's workspace permissions; unauthorized actions are hidden rather than disabled.

#### Acceptance Criteria

##### Dashboard Load

Given a logged-in workspace member, when they open the dashboard, then they see an overview of their work and workspace activity.

##### Personal Tasks

Given assigned issues, when the dashboard loads, then those issues appear in the personal overview.

##### Blocked Work

Given the current user has blocked assigned issues, when the dashboard loads, then those issues are clearly identified as blocked.

##### Activity Feed

Given recent workspace activity, when the dashboard loads, then the latest activity is displayed first.

##### Quick Actions

Given a user with permission, when they select a quick action, then the corresponding workflow begins immediately.

Given an Owner or Admin views Dashboard actions, then Create Project and Create Cycle are available alongside the actions shared by all roles.

Given a Member views Dashboard or empty-state actions, then Create Project and Create Cycle are not displayed.

#### Edge Cases

- Workspace with no projects.
- Workspace with no issues.
- User has no assigned work.
- No active development cycle.
- Empty activity feed.
- Recently archived items.

#### Future Enhancements

- Customizable dashboard widgets.
- Team performance insights.
- Burndown and velocity charts.
- Calendar summary.
- Personalized recommendations.
- AI-generated daily summaries.

### 5.5 Issues

#### Overview

Issues represent individual units of work within a workspace. They are used to track bugs, features, tasks, improvements, and other development work from creation to completion.

#### Goals

- Capture all development work.
- Provide a clear workflow from backlog to completion.
- Keep issue information organized and searchable.
- Enable collaboration around work items.

#### User Stories

- As a user, I want to create an issue so work can be tracked.
- As a user, I want to update an issue as work progresses.
- As a team lead, I want to assign issues to team members.
- As a developer, I want to know the priority of my work.
- As a team member, I want to discuss work within an issue.

#### Functional Requirements

##### Issue Creation

- Users can create an issue.
- Every issue must have a title.
- Issues may include a description.
- Issue priority is optional during creation.
- Issues are automatically assigned a unique identifier.
- New issues are created in the Backlog by default.
- New issues use No Priority when no priority is selected.

##### Issue Details

Each issue supports:

- Title
- Description
- Status
- Priority
- Assignee
- Project
- Cycle
- Labels
- Due date
- Blocked flag
- Blocked reason (optional)
- Creator
- Created date
- Last updated date

##### Issue Management

Users can:

- Edit issue details.
- Change issue status.
- Assign or reassign issues.
- Add or remove labels.
- Associate an issue with a project.
- Associate an issue with a cycle.
- Set or update a due date.
- Mark an issue as blocked with an optional reason.
- Clear an issue's blocked flag and reason.
- Archive an issue.
- Restore an archived issue.
- Delete an issue (subject to permissions).

##### Issue Workflow

Issues move through predefined workflow states:

- Backlog
- Todo
- In Progress
- Done

The system records status changes in the issue history.

Blocked is an independent issue flag, not a workflow status. Marking or clearing the flag does not change the issue's Backlog, Todo, or In Progress status.

##### Issue Relationships

- Every issue belongs to one workspace.
- An issue may belong to one project.
- An issue may belong to one cycle.
- An issue may have one assignee.
- An issue may have multiple labels.
- Projects and cycles are independent workspace entities.
- Any relationship shown between a project and a cycle is derived from issues that belong to both.

##### Issue Discovery

Users can:

- Search issues.
- Filter issues.
- Sort issues.
- View issue details.
- View issue history.

#### Business Rules

- Every issue belongs to exactly one workspace.
- Every issue has one current status.
- Every issue has a priority value; when the creator does not select one, the value defaults to No Priority.
- Archived issues are read-only.
- Archived issues may be restored by members with permission to archive issues.
- Restoring an issue returns it to the workflow status and blocked state it held before archiving.
- New issues are unblocked by default.
- Only issues in Backlog, Todo, or In Progress may be marked as blocked.
- A blocked issue may include an optional blocked reason.
- Moving a blocked issue to Done automatically clears its blocked flag and reason.
- Returning a Done issue to an active status does not restore its previous blocked state.
- Blocked state does not affect Project or Cycle completion calculations.
- Completed issues remain searchable.
- Issue identifiers are unique within a workspace.
- Only authorized members can delete issues.
- Changes to an issue are recorded in its activity history.

#### Acceptance Criteria

##### Create Issue

Given a workspace member, when they create an issue with a valid title, then the issue is added to the workspace backlog and uses No Priority if no priority was selected.

##### Update Status

Given an existing issue, when its status is changed, then the new status is saved and recorded in the issue history.

##### Assign Issue

Given a workspace member, when they assign an issue, then the assignee is updated successfully.

##### Archive Issue

Given an existing issue, when it is archived, then it no longer appears in active issue lists.

##### Restore Issue

Given an archived issue and an authorized member, when the issue is restored, then it returns to its previous workflow status and blocked state and becomes editable again.

##### Mark Issue as Blocked

Given an active unfinished issue, when an authorized member marks it as blocked with an optional reason, then the blocked state is displayed and recorded in issue history without changing workflow status.

##### Complete Blocked Issue

Given a blocked issue, when its status changes to Done, then its blocked flag and reason are cleared and the changes are recorded in issue history.

##### Search Issues

Given matching issues exist, when a user performs a search, then relevant issues are returned.

#### Edge Cases

- Empty issue title.
- Extremely long title or description.
- Deleted assignee.
- Archived project linked to an issue.
- Archived cycle linked to an issue.
- Duplicate labels.
- Invalid due date.
- Simultaneous edits by multiple users.
- Attempt to edit an archived issue.
- Attempt to mark a Done or archived issue as blocked.
- Empty or extremely long blocked reason.

#### Future Enhancements

- Parent-child issues.
- Issue dependencies.
- Subtasks.
- Custom issue types.
- Recurring issues.
- Issue templates.
- Watchers.
- Estimated effort/story points.
- Rich text editing.
- File attachments.

### 5.6 Projects

#### Overview

Projects organize related issues into larger initiatives with a shared objective. They provide teams with visibility into progress, ownership, and timelines for delivering features or milestones.

#### Goals

- Organize related work into meaningful initiatives.
- Track progress toward project goals.
- Provide visibility into project status.
- Help teams plan and deliver features efficiently.

#### User Stories

- As a product manager, I want to create a project for a new feature.
- As a team lead, I want to group related issues under a project.
- As a developer, I want to understand how my work contributes to the project.
- As a stakeholder, I want to monitor project progress.
- As an Owner or Admin, I want to assign clear ownership for each Project.

#### Functional Requirements

##### Project Creation

- Users with permission can create a project.
- Every project must have a name.
- A project name must be unique within its workspace.
- Projects may include a description.
- Projects may have a start date.
- Projects may have a target completion date.
- Every project has one Project Owner.
- The Project creator becomes the Project Owner by default.

##### Project Details

Each project supports:

- Name
- Description
- Status
- Owner
- Start date
- Target date
- Progress
- Associated issues
- Created date
- Last updated date

##### Project Management

Users with permission can:

- Edit project information.
- Archive a project.
- Restore an archived project.
- Delete a project.
- Transfer a non-archived Project to another member when acting as a Workspace Owner or Admin.
- Update the project between Planned, Active, and Completed.

##### Project Organization

Users can:

- Add issues to a project.
- Remove issues from a project.
- View all issues within a project.
- Search and filter project issues.
- Navigate from a project to any associated issue.

##### Project Progress

The system should:

- Display the total number of issues.
- Display completed and remaining issues.
- Display overall project progress.
- Update project progress automatically as issue statuses change.

##### Project Status

Projects support the following operational statuses:

- Planned
- Active
- Completed

Authorized editors can switch freely between these operational statuses in any direction without confirmation.

Projects also support an Archived lifecycle state:

- Archived

Archived is not available in the status control. A Project enters or leaves Archived only through the separate Archive or Restore action, each of which requires confirmation.

#### Business Rules

- Every project belongs to exactly one workspace.
- Every Project Owner is a current member of the Project's workspace.
- Project ownership grants no additional workspace or resource permissions.
- Workspace Owners and Admins may transfer a non-archived Project to any other current Owner, Admin, or Member.
- If a Project Owner leaves or is removed, their Projects transfer automatically to the Workspace Owner in the same operation.
- Project-name uniqueness is scoped to a workspace; different workspaces may use the same project name.
- Project-name comparison is case-insensitive after trimming leading and trailing whitespace.
- Archived projects continue to reserve their names; permanent deletion releases a name for reuse.
- Creating or renaming a project is blocked when the normalized name conflicts with another Project in the workspace.
- A project can contain multiple issues.
- An issue can belong to only one project.
- Planned, Active, and Completed can be selected freely through the Project status control.
- The Project status control never includes Archived.
- Archived projects are read-only.
- Archived projects may be restored by users with permission to archive projects.
- Restoring a project returns it to the status it held before archiving.
- Deleting a project is permanent and does not delete its issues.
- When a project is deleted, every associated issue remains in the workspace and its project assignment is cleared automatically.
- Project deletion and issue unassignment occur atomically; if either operation fails, neither change is saved.
- Removing an issue from a project does not delete the issue.

#### Acceptance Criteria

##### Create Project

Given a user with permission, when they provide the required information and a name unique within the workspace, then a new project is created with that user as its Project Owner.

##### Rename Project

Given an authorized editor and a non-archived project, when they enter a name that does not conflict with another Project in the workspace, then the new name is saved.

##### Transfer Project Ownership

Given a Workspace Owner or Admin and a non-archived Project, when they select any other current workspace member, then that member becomes Project Owner without any change to their workspace role or permissions.

##### Add Issue to Project

Given an existing project and issue, when the issue is added, then it appears in the project's issue list.

##### Track Progress

Given project issues change status, when progress is recalculated, then the project reflects the updated completion percentage.

##### Update Project Status

Given a non-archived project and an authorized editor, when they select Planned, Active, or Completed, then the project changes directly to that status without confirmation.

##### Archive Project

Given a non-archived project and an authorized user, when they confirm Archive, then its operational status is stored and it becomes read-only in the Archived Projects list.

##### Restore Project

Given an archived project and an authorized user, when they confirm Restore, then it returns to its stored Planned, Active, or Completed status and becomes editable again.

##### Delete Project

Given an existing project and an authorized user, when deletion is confirmed, then the project is permanently removed, its name becomes available for reuse in the workspace, and all associated issues remain without a project assignment.

#### Edge Cases

- Project created without issues.
- All issues removed from a project.
- Project Owner leaves or is removed while owning active or Archived Projects.
- Archived issue remains associated with a project.
- Project reaches its target date with unfinished issues.
- Attempt to create or rename a project with a case-insensitive, whitespace-trimmed name conflict.
- Attempt to modify an archived project.
- Project deleted while it has associated issues.
- Project deletion fails while issue assignments are being cleared.

#### Future Enhancements

- Project templates.
- Milestones.
- Project dependencies.
- Roadmaps.
- Project health indicators.
- Custom project fields.
- Cross-project reporting.
- Budget and resource tracking.

### 5.7 Cycles

#### Overview

Cycles are fixed development periods used to plan, organize, and track work. They help teams focus on a defined set of issues and measure progress toward short-term goals.

#### Goals

- Organize work into time-boxed iterations.
- Help teams prioritize deliverables.
- Improve visibility into sprint progress.
- Keep teams focused on current work.

#### User Stories

- As a team lead, I want to create a development cycle.
- As a team lead, I want to assign issues to a cycle.
- As a developer, I want to know what work belongs to the current cycle.
- As a team member, I want to track cycle progress.

#### Functional Requirements

##### Cycle Creation

- Users with permission can create a cycle.
- Every cycle must have a name.
- A cycle name must be unique within its workspace.
- Every cycle must have a start date.
- Every cycle must have an end date.
- A cycle may include a goal or description.
- Every new cycle starts in Planned status.
- A new cycle's date range must not overlap another non-archived cycle in the same workspace.

##### Cycle Management

Users with permission can:

- Edit cycle details.
- Start a cycle.
- Complete a cycle.
- Reopen a completed cycle.
- Archive a cycle.
- Restore an archived cycle.
- Delete a future Planned cycle before its start date.

##### Issue Planning

Users can:

- Add issues to a cycle.
- Remove issues from a cycle.
- Move issues between cycles.
- View all issues assigned to a cycle.

##### Cycle Progress

The system should:

- Display total issues.
- Display completed issues.
- Display remaining issues.
- Display overall completion progress.
- Update progress automatically as issue statuses change.

##### Cycle Status

Cycles support the following statuses:

- Planned
- Active
- Completed
- Archived

Status is changed only through the Start, Complete, Reopen, Archive, and Restore actions. Users cannot select an arbitrary status.

Allowed transitions are:

- Planned to Active through Start.
- Active to Completed through Complete.
- Completed to Active through Reopen.
- Planned or Completed to Archived through Archive.
- Archived to its stored pre-archive status through Restore.

Only one cycle can be Active within a workspace at a time. An Active cycle cannot be archived; it must be completed first.

#### Business Rules

- Every cycle belongs to one workspace.
- Cycle-name uniqueness is scoped to a workspace; different workspaces may use the same cycle name.
- Cycle-name comparison is case-insensitive after trimming leading and trailing whitespace.
- Archived cycles continue to reserve their names; permanent deletion releases a name for reuse.
- Creating or renaming a cycle is blocked when the normalized name conflicts with another Cycle in the workspace.
- An issue can belong to only one cycle at a time.
- Date ranges of non-archived cycles cannot overlap within the same workspace.
- Start and end dates are inclusive, so a following cycle must start after the preceding cycle's end date.
- Editing cycle dates must preserve the no-overlap rule.
- Completed cycles become read-only unless reopened.
- Completing a cycle does not automatically complete unfinished issues.
- Archived cycles remain available for historical reference.
- Archived cycles may be restored by users with permission to manage cycles.
- Archived cycles do not block scheduling, but restoration is blocked when the restored date range would overlap a non-archived cycle.
- A cycle archived while Planned or Completed returns to its stored pre-archive status when restored.
- Reopening a Completed cycle is blocked when another cycle is Active or when the date range conflicts with another non-archived cycle.
- Deleting a future Planned cycle permanently removes it and unassigns its issues from the cycle without deleting those issues.

#### Acceptance Criteria

##### Create Cycle

Given a user with permission, when they provide valid cycle information with a name unique within the workspace and dates that do not overlap another non-archived cycle, then a new Planned cycle is created.

##### Rename Cycle

Given an authorized editor and an editable cycle, when they enter a name that does not conflict with another Cycle in the workspace, then the new name is saved.

##### Assign Issue to Cycle

Given an existing issue, when it is assigned to a cycle, then it appears in that cycle's issue list.

##### Start Cycle

Given a Planned cycle and no other Active cycle, when an authorized user starts it, then its status changes to Active.

##### Complete Cycle

Given an active cycle, when it is completed, then its status changes to Completed and no further edits are allowed.

##### Restore Cycle

Given an Archived cycle and an authorized user, when its dates do not overlap another non-archived cycle, then it returns to its stored Planned or Completed status.

##### Reopen Cycle

Given a Completed cycle, when no other cycle is Active and its dates do not conflict, then an authorized user can reopen it as Active.

##### Delete Cycle

Given a future Planned cycle, when an authorized user confirms deletion, then the cycle is permanently deleted and its issues become unassigned from any cycle.

#### Edge Cases

- Cycle created with no issues.
- Cycle reaches its end date before completion.
- Issue moved out of an active cycle.
- Attempt to activate a second cycle while another is active.
- Attempt to create, restore, reopen, or reschedule a cycle with overlapping dates.
- Attempt to create or rename a cycle with a case-insensitive, whitespace-trimmed name conflict.
- Attempt to archive an Active cycle before completing it.
- Attempt to edit a completed cycle.
- Future Planned cycle deleted while it still contains issues.

#### Future Enhancements

- Automatic recurring cycles.
- Carry unfinished issues into the next cycle.
- Sprint velocity reporting.
- Burndown charts.
- Capacity planning.
- Team workload balancing.

### 5.8 Comments

#### Overview

Comments allow workspace members to discuss work directly within an issue, keeping conversations organized and attached to the relevant context.

#### Goals

- Enable collaboration around issues.
- Keep discussions contextual.
- Maintain a history of decisions and updates.
- Reduce the need for external communication tools.

#### User Stories

- As a developer, I want to ask questions about an issue.
- As a reviewer, I want to provide feedback on an issue.
- As a team member, I want to mention a colleague to get their attention.
- As a user, I want to edit or remove my comments if needed.

#### Functional Requirements

##### Comment Creation

- Members can add comments to an issue.
- Comments support plain text and basic formatting.
- Comments are displayed in chronological order.
- Every comment displays its author and creation time.

##### Comment Management

Users can:

- Edit their own comments.
- Delete their own comments.
- View the edit history indicator for modified comments.

##### Mentions

- Users can mention workspace members using `@`.
- Mentioned users receive a notification.
- Mentions link to the referenced member.

##### Comment History

- Comments remain associated with their issue.
- Deleted comments are removed from the conversation.
- Edited comments display that they have been modified.

#### Business Rules

- Comments belong to exactly one issue.
- Only workspace members can comment.
- Archived issues cannot receive new comments.
- Users can only edit or delete their own comments; Owner and Admin roles provide no moderation override in the MVP.
- Comments are ordered chronologically.

#### Acceptance Criteria

##### Add Comment

Given an active issue, when a member submits a comment, then the comment appears in the issue discussion.

##### Edit Comment

Given an existing comment, when its author edits it, then the updated content is displayed and marked as edited.

##### Delete Comment

Given an existing comment, when its author confirms deletion, then the comment is removed from the issue.

##### Mention User

Given a workspace member, when they are mentioned in a comment, then they receive a notification.

##### Archived Issue

Given an archived issue, when a user attempts to comment, then the action is prevented.

#### Edge Cases

- Empty comment submission.
- Very long comments.
- Mentioning a non-existent user.
- Mentioning a user who has left the workspace.
- Editing a deleted comment.
- Attempting to edit or delete another user's comment.
- Simultaneous edits by multiple users.

#### Future Enhancements

- Emoji reactions.
- Rich text editor.
- File attachments.
- Code blocks with syntax highlighting.
- Comment threads and replies.
- Comment pinning.

### 5.9 Notifications

#### Overview

Notifications inform users about relevant activity within their workspace, helping them stay updated without constantly checking the application.

#### Goals

- Keep users informed about important events.
- Surface actions that require attention.
- Reduce the need to manually monitor Issue assignments and mentions.
- Support timely collaboration.

#### User Stories

- As a user, I want to know when I'm assigned an issue.
- As a user, I want to know when someone mentions me.
- As a workspace member, I want to review notifications I've missed.

#### Functional Requirements

##### Notification Types

The system generates notifications for:

- Issue assignment or reassignment, delivered to the new assignee.
- Mentions in comments, delivered to each mentioned workspace member.

##### Notification Center

Users can:

- View all notifications.
- View unread notifications.
- Mark a notification as read.
- Mark all notifications as read.
- Navigate directly to the related item.

##### Notification Status

Each notification includes:

- Type
- Message
- Related item
- Timestamp
- Read/Unread status

##### Notification Management

Users can:

- Mark notifications as read.
- Mark all notifications as read.
- Delete individual notifications.
- Clear all notifications.

#### Business Rules

- Notifications are private to the recipient.
- Notifications belong to one user.
- Only Issue assignment or reassignment and comment mentions generate notifications in the MVP.
- Read notifications remain accessible until deleted.
- Deleted notifications cannot be restored.
- Opening a notification navigates to the related Issue when it still exists.

#### Acceptance Criteria

##### Receive Assignment Notification

Given an Issue is assigned or reassigned, when the change is saved, then the new assignee receives a notification linked to that Issue.

##### Receive Mention Notification

Given a workspace member is mentioned in a comment, when the comment is submitted, then the mentioned member receives a notification linked to that Issue.

##### Mark as Read

Given an unread notification, when the user marks it as read, then its status changes to read.

##### Open Notification

Given a notification references an existing Issue, when the user selects it, then they are taken to the Issue Details page.

##### Clear Notifications

Given existing notifications, when the user clears them, then they are removed from the notification list.

#### Edge Cases

- Related Issue is deleted.
- User is mentioned multiple times in the same comment.
- Notification references an Archived Issue.
- Large number of unread notifications.
- Duplicate events occurring in quick succession.

#### Future Enhancements

- Email notifications.
- Push notifications.
- Notification preferences.
- Notification grouping.
- Notification snoozing.
- Real-time desktop notifications.
- Digest summaries.
- Notifications for Issue unassignment or status changes, Workspace invitations, Project ownership changes, and Cycle changes.

### 5.10 Search & Filters

#### Overview

Search and Filters enable users to quickly locate issues, projects, and other workspace resources by searching, filtering, and sorting data.

#### Goals

- Help users find information quickly.
- Reduce time spent navigating the workspace.
- Support common engineering workflows.
- Make large workspaces easy to manage.

#### User Stories

- As a developer, I want to quickly find an issue by its title.
- As a team lead, I want to filter issues by status and assignee.
- As a product manager, I want to view only issues related to a project.
- As a user, I want to sort results to focus on what matters.

#### Functional Requirements

##### Search

Users can search for:

- Issues
- Projects
- Cycles
- Members

Search should match relevant information such as names, titles, and descriptions.

##### Filtering

Users can filter issues by:

- Status
- Priority
- Assignee
- Project
- Cycle
- Labels
- Creator
- Due date
- Blocked state

Users can apply multiple filters simultaneously.

##### Sorting

Users can sort results by:

- Created date
- Updated date
- Due date
- Priority
- Title

Users can change the sort order between ascending and descending where applicable.

##### Saved Views

Users can:

- Save frequently used filter combinations.
- Rename saved views.
- Delete saved views.
- Set a saved view as the default.

##### Search Results

The system should:

- Display matching results.
- Display the number of matching results.
- Show a message when no results are found.
- Update results whenever search or filter criteria change.

#### Business Rules

- Search only returns data from the current workspace.
- Users only see resources they have permission to access.
- Multiple filters are combined together.
- Saved views are private to the user.

#### Acceptance Criteria

##### Search

Given matching resources exist, when a user enters a search query, then relevant results are displayed.

##### Apply Filters

Given available filters, when one or more filters are applied, then only matching results are shown.

##### Save View

Given an active filter configuration, when the user saves it, then the saved view is available for future use.

##### No Results

Given no matching resources exist, when a search or filter is applied, then an appropriate empty state is displayed.

#### Edge Cases

- Empty search query.
- Search returns no results.
- Invalid filter combinations.
- Saved view references deleted resources.
- Filtering archived resources.
- Very large result sets.

#### Future Enhancements

- Global search across all workspaces.
- Natural language search.
- Recent searches.
- Search suggestions.
- Advanced query syntax.
- Shared saved views.
- Search history.

### 5.11 Settings

#### Overview

Settings are divided into global Account Settings and workspace-specific Settings. Account Settings remain available without an active workspace; Workspace Settings apply only to the selected workspace.

#### Goals

- Provide clear, separate locations for account and workspace configuration.
- Allow users to manage their profile, security, and appearance.
- Allow workspace administrators to manage workspace settings.
- Keep administrative tasks organized and accessible.

#### User Stories

- As a user, I want to update my profile information.
- As a user, I want to change my password.
- As a user, I want to select my preferred interface theme.
- As a workspace owner, I want to manage workspace settings.
- As a workspace owner, I want to manage member roles.

#### Functional Requirements

##### Profile Settings

Users can:

- Update their display name.
- Update their profile picture.
- Update their email address.
- View their account information.

##### Security Settings

Users can change their password after providing their current password.

Changing an account email address also requires the user's current password.

##### Appearance Settings

Users can select a theme:

- Light
- Dark
- System

##### Workspace Settings

Workspace owners can:

- Update the workspace name.
- Update the workspace icon.
- Archive the workspace.
- Open an Archived workspace to permanently delete it.
- View workspace information.

##### Member Management

All workspace members can view the member directory.

Workspace Owners can:

- Change Member and Admin roles.
- Remove Members and Admins.
- Transfer workspace ownership to an existing Member or Admin.

Workspace Admins can:

- Remove Members.
- Invite users as Members.

##### Danger Zone

Workspace owners can:

- Archive the workspace.
- Permanently delete the workspace after it has been archived.

Archiving requires confirmation. Permanent deletion requires the Owner to type the exact workspace name before confirming.

#### Business Rules

- Users can only modify their own account settings.
- Authenticated users can access Account Settings without selecting a workspace.
- Account changes apply across every workspace the user belongs to.
- Email addresses must remain valid and unique.
- Changing an email address requires the current password.
- Only workspace owners can modify workspace settings.
- Only authorized users can manage members.
- Every workspace has exactly one Owner.
- Destructive actions require confirmation.

#### Acceptance Criteria

##### Update Profile

Given a logged-in user, when they save valid profile changes, then the updated information is reflected across their account and every workspace they belong to.

##### Change Email

Given a logged-in user provides their current password and an unused valid email address, when they confirm the change, then the new email becomes their account email across all workspaces.

##### Change Password

Given a logged-in user, when they provide valid credentials, then the password is updated successfully.

##### Change Theme

Given a logged-in user, when they choose Light, Dark, or System, then the selected theme is saved to their account and applied throughout the product.

##### Update Workspace

Given a workspace owner, when they modify workspace information, then the changes are visible to all workspace members.

##### Delete Workspace

Given a workspace owner and an Archived workspace, when they enter the exact workspace name and confirm deletion, then all workspace-scoped data and memberships are permanently removed without deleting member user accounts.

#### Edge Cases

- Multiple workspaces visible to a user share the same display name.
- Invalid email address.
- User attempts to change restricted settings.
- Owner attempts to leave before transferring ownership.
- Owner attempts to delete a workspace before archiving it.
- Workspace name confirmation does not match exactly.
- Attempt to perform destructive actions without confirmation.

#### Future Enhancements

- Custom branding.
- Language and localization settings.
- Notification preferences.
- Active-session management.
- Workspace domains.
- API keys.
- Webhooks.
- Integration management.
- Data export.
- Data import.
- Audit history.
- Billing and subscription settings.

## 6. Permissions & Roles

### Overview

Shipyard uses role-based access control (RBAC) to determine what actions members can perform within a workspace.

The MVP includes three roles:

- **Owner:** Full administrative access.
- **Admin:** Manages day-to-day team operations.
- **Member:** Collaborates on projects and issues.

### Permission Matrix

| Permission | Owner | Admin | Member |
| --- | :---: | :---: | :---: |
| View workspace | ✅ | ✅ | ✅ |
| Update workspace | ✅ | ❌ | ❌ |
| Archive or restore workspace | ✅ | ❌ | ❌ |
| Delete archived workspace | ✅ | ❌ | ❌ |
| View member directory | ✅ | ✅ | ✅ |
| Invite users as Members | ✅ | ✅ | ❌ |
| Invite users as Admins | ✅ | ❌ | ❌ |
| Remove Members | ✅ | ✅ | ❌ |
| Remove Admins | ✅ | ❌ | ❌ |
| Change Member/Admin roles | ✅ | ❌ | ❌ |
| Transfer Workspace ownership to a Member or Admin | ✅ | ❌ | ❌ |
| Create projects | ✅ | ✅ | ❌ |
| Edit projects | ✅ | ✅ | ✅ |
| Transfer Project ownership | ✅ | ✅ | ❌ |
| Archive or restore projects | ✅ | ✅ | ❌ |
| Delete projects | ✅ | ✅ | ❌ |
| Create issues | ✅ | ✅ | ✅ |
| Edit issues | ✅ | ✅ | ✅ |
| Archive or restore issues | ✅ | ✅ | ✅ |
| Delete issues | ✅ | ✅ | ❌ |
| Create cycles | ✅ | ✅ | ❌ |
| Manage cycle lifecycle, including deletion | ✅ | ✅ | ❌ |
| Comment on issues | ✅ | ✅ | ✅ |
| Edit or delete own comments | ✅ | ✅ | ✅ |
| Edit or delete another user's comments | ❌ | ❌ | ❌ |
| View dashboard | ✅ | ✅ | ✅ |
| Search workspace | ✅ | ✅ | ✅ |
| Manage workspace settings | ✅ | ❌ | ❌ |

### Role Definitions

#### Owner

Responsible for the workspace and its administration.

Can:

- Manage workspace settings.
- Manage members.
- Transfer Workspace ownership to an existing Member or Admin, becoming an Admin afterward.
- Transfer a non-archived Project to any other current workspace member.
- Archive an active workspace or restore or delete it after archival.
- Perform all Admin and Member actions.

#### Admin

Responsible for managing development work.

Can:

- Manage projects.
- Transfer a non-archived Project to any other current workspace member.
- Manage issues.
- Manage cycles.
- Invite users as Members.
- Remove Members.
- Perform all Member actions.

Cannot:

- Delete the workspace.
- Modify workspace settings.
- Change member roles.
- Transfer Workspace ownership.

#### Member

Responsible for contributing to the team's work.

Can:

- Create and update issues.
- Contribute to projects.
- Comment on issues.
- View dashboards.
- Search workspace data.

Cannot:

- Create projects.
- Manage workspace administration.
- Manage members or roles.
- Delete projects.
- Create or manage cycles.

### Business Rules

- Every workspace has exactly one Owner.
- Permissions are evaluated within each workspace.
- Users may have different roles in different workspaces.
- Unauthorized actions are rejected.
- Role changes take effect immediately.
- Feature-specific permissions follow this workspace permission matrix unless explicitly stated otherwise.

### Acceptance Criteria

#### Permission Check

Given a member attempts an action, when they have the required permission, then the action succeeds.

#### Unauthorized Action

Given a member attempts an action, when they lack permission, then the action is denied.

### Edge Cases

- Owner attempts to leave before transferring ownership.
- Role changed while the user is active.
- Removed member attempts to perform an action.
- Workspace ownership transferred during an active session.

### Future Enhancements

- Custom roles.
- Fine-grained permissions.
- Permission groups.
- Organization-level roles.
- Resource-specific permissions.

## 7. Business Rules

### Overview

Business Rules define the constraints and behaviors that govern how Shipyard operates. These rules apply across the product and ensure consistent behavior regardless of where an action is performed.

### 7.1 Workspace Rules

- Every workspace has exactly one Owner.
- Every workspace has one immutable, unique internal identifier.
- Workspace names may duplicate and are never used as identifiers.
- A user may belong to multiple workspaces.
- User roles are assigned independently for each workspace.
- All projects, issues, cycles, and members belong to exactly one workspace.
- Users can only access workspaces they are members of.
- An active workspace must be archived before permanent deletion.
- Only the Owner can permanently delete an Archived workspace, and the exact workspace name must be entered to confirm.
- Deleting a workspace permanently removes all workspace-scoped data and memberships without deleting user accounts.

### 7.2 Member Rules

- Every member has exactly one role within a workspace.
- A member can only hold one role at a time.
- Role changes take effect immediately.
- All members can view the workspace member directory.
- Owners can invite users as Members or Admins and can remove Members or Admins.
- Admins can invite and remove Members only.
- Only the Owner can change Member/Admin roles or transfer Workspace ownership.
- Removed members immediately lose access to the workspace.
- Projects owned by a removed or departing Member or Admin transfer automatically to the Workspace Owner.
- Members and Admins may leave a workspace voluntarily.
- Ownership transfer must update the recipient to Owner and the transferring Owner to Admin atomically.
- The Owner cannot leave or be removed until ownership is transferred.

### 7.3 Project Rules

- Every project belongs to exactly one workspace.
- Every project has exactly one Project Owner, and the creator is the initial Project Owner.
- Project ownership grants no additional permissions.
- Workspace Owners and Admins may transfer a non-archived Project to any other current workspace member.
- Projects owned by a departing or removed member transfer automatically to the Workspace Owner.
- Project names are unique within a workspace after case-insensitive comparison and trimming surrounding whitespace.
- Archived projects reserve their names until permanently deleted.
- Projects may contain zero or more issues.
- Projects do not directly contain or own cycles.
- Members can contribute to projects but cannot create or delete them.
- Project editors may switch freely between Planned, Active, and Completed.
- Archived is available only through confirmed Archive and Restore actions, not through the Project status control.
- Archived projects become read-only until restored.
- Restoring a project returns it to its previous status.
- Deleting a project permanently removes only the project and automatically clears that project from its issues.
- Project deletion and issue unassignment must succeed or fail as one operation.

### 7.4 Issue Rules

- Every issue belongs to exactly one workspace.
- Every issue may belong to one project.
- Every issue may belong to one cycle.
- Every issue has exactly one status.
- Every issue has exactly one creator.
- Every issue may have one assignee.
- Every issue has a priority value, defaulting to No Priority when omitted during creation.
- Issues retain their history after updates.
- Deleted issues cannot be recovered.

### 7.5 Cycle Rules

- Every cycle belongs to exactly one workspace.
- Cycle names are unique within a workspace after case-insensitive comparison and trimming surrounding whitespace.
- Archived cycles reserve their names; permanently deleting an eligible future Planned cycle releases its name.
- Cycles do not directly belong to projects.
- Only one cycle may be active within a workspace at any time.
- Non-archived cycle date ranges may never overlap within a workspace.
- An issue may belong to only one cycle.
- Completing a cycle does not automatically move unfinished issues.
- Archived cycles become read-only until restored.
- Active cycles must be completed before they can be archived.
- Restoring or reopening a cycle must preserve both the one-active-cycle and no-overlap rules.

### 7.6 Comment Rules

- Every comment belongs to one issue.
- Every comment has one author.
- Users may edit or delete only their own comments; workspace roles do not override comment authorship.
- Mentioning a user generates a notification.

### 7.7 Notification Rules

- Notifications are private to the recipient.
- Notifications belong to one user.
- Notifications remain available until deleted.
- Opening a notification does not remove it.
- Marking a notification as read only changes its status.

### 7.8 Search Rules

- Search is limited to the current workspace.
- Users only see resources they have permission to access.
- Multiple filters are combined when applied together.
- Saved views are private to the user.

### 7.9 General Rules

- Archived resources cannot be modified until restored.
- The Global Create Menu is available only when an active workspace is selected.
- The Global Create Menu offers Create Issue to Owners, Admins, and Members.
- The Global Create Menu offers Create Project and Create Cycle only to Owners and Admins.
- Creation actions the current user lacks permission to perform are omitted from the Global Create Menu rather than shown disabled.
- Users who can archive a resource can also restore it.
- Before archiving a resource, the system records the state required to restore it correctly.
- Restoration preserves the resource's data and history.
- Permanently deleted resources cannot be restored.
- Unauthorized actions are rejected.
- System timestamps are recorded automatically.
- All user-generated content is associated with its creator.
- Destructive actions require explicit confirmation.
- Changes should be reflected consistently across the workspace.

## 8. Non-Functional Requirements

### Overview

Non-Functional Requirements define the quality attributes of Shipyard. These requirements ensure the product is secure, reliable, performant, accessible, and scalable while providing a consistent user experience.

### 8.1 Performance

#### Requirements

- The application should feel responsive during normal use.
- Most user interactions should provide immediate visual feedback.
- Search and filtering should return results quickly.
- Navigation between pages should be smooth.
- Large workspaces should remain usable without noticeable slowdowns.
- Background operations should not interrupt the user experience.

### 8.2 Reliability

#### Requirements

- The application should handle unexpected failures gracefully.
- User data should remain consistent after operations.
- Failed actions should display clear error messages.
- Users should be able to safely retry failed operations.
- The system should prevent duplicate actions caused by repeated user input.

### 8.3 Security

#### Requirements

- Users must be authenticated before accessing protected resources.
- Authorization must be enforced for all protected actions.
- Users may only access data belonging to workspaces they are members of.
- Sensitive user information must be protected.
- Destructive actions require explicit confirmation.
- User sessions should be managed securely.

### 8.4 Scalability

#### Requirements

- The system should support growth in users, workspaces, and projects.
- Performance should remain acceptable as data volume increases.
- The product should support future feature expansion without major redesign.
- The architecture should accommodate additional integrations and modules.

### 8.5 Availability

#### Requirements

- The application should be available whenever users need it.
- Temporary service interruptions should be communicated clearly.
- Users should receive meaningful feedback when services are unavailable.

### 8.6 Usability

#### Requirements

- The interface should be simple and intuitive.
- Common actions should require minimal effort.
- Navigation should remain consistent across the product.
- Empty states should guide users toward the next action.
- Error messages should clearly explain what went wrong.
- Confirmation messages should acknowledge successful actions.

### 8.7 Accessibility

#### Requirements

- The application should be navigable using a keyboard.
- Interactive elements should be clearly distinguishable.
- Color should not be the only indicator of information.
- Text should remain readable across supported themes.
- Interface components should support assistive technologies where applicable.

### 8.8 Compatibility

#### Requirements

- The application should support modern desktop browsers.
- The interface should adapt to common screen sizes.
- Users should have a consistent experience across supported browsers.
- The product should remain functional on tablets, with mobile support considered in future releases.

### 8.9 Maintainability

#### Requirements

- Product behavior should remain consistent as new features are added.
- Features should be modular to simplify future development.
- Documentation should remain aligned with product behavior.
- Changes to one feature should minimize unintended effects on others.

### 8.10 Observability

#### Requirements

- System errors should be recorded for investigation.
- Significant user actions should be traceable where appropriate.
- Application health should be monitorable.
- Operational issues should be diagnosable without impacting users.

### 8.11 Data Integrity

#### Requirements

- Data should remain accurate and consistent after operations.
- Relationships between resources should remain valid.
- Invalid or incomplete data should not be accepted.
- Operations should avoid creating duplicate or conflicting records.

### Acceptance Criteria

#### Performance

Given normal application usage, when users perform common actions, then the application remains responsive and usable.

#### Security

Given an unauthorized user, when they attempt to access protected resources, then access is denied.

#### Reliability

Given an operation fails, when the failure occurs, then the user receives clear feedback and data remains consistent.

#### Accessibility

Given a user navigates without a mouse, when they interact with the application, then all primary functionality remains accessible.

### Future Enhancements

- Offline support.
- Progressive Web App (PWA) capabilities.
- Multi-region deployment.
- Advanced monitoring and analytics.
- Localization and internationalization.
- Configurable performance and retention policies.

## 9. Release Scope

### Overview

This section defines the functionality planned for the initial release (MVP), identifies features deferred to future releases, and clarifies what is outside the scope of Shipyard.

### 9.1 MVP Scope (Version 1.0)

The initial release focuses on providing a complete project management experience for small software teams.

#### Authentication

- User registration
- User login
- Logout
- Password management
- Session management

#### Workspace Management

- Create workspace
- Update workspace
- Archive and restore workspace
- Permanently delete archived workspace
- Transfer workspace ownership
- Invite members
- Member roles
- Workspace switching

#### Issue Management

- Create issues
- Update issues
- Archive and restore issues
- Delete issues
- Assign issues
- Issue priorities
- Issue statuses
- Labels
- Due dates
- Search and filtering
- Blocked issue visibility

#### Project Management

- Create projects
- Update projects
- Transfer Project ownership
- Archive and restore projects
- Delete projects and unassign their issues
- Track project progress

#### Cycle Management

- Create cycles
- Start, complete, and reopen cycles
- Enforce non-overlapping cycle schedules
- Archive and restore cycles
- Delete future planned cycles
- Assign issues to cycles
- Cycle progress tracking

#### Collaboration

- Comments
- User mentions
- Notifications

#### Dashboard

- Personal overview
- Workspace overview
- Activity feed
- Quick actions

#### Settings

- Profile management
- Email management
- Password management
- Theme preference
- Workspace settings
- Member management

### 9.2 Post-MVP Scope

These features are planned for future releases but are not part of Version 1.0.

#### Planning

- Kanban Board
- Timeline View
- Calendar View
- Roadmaps
- Milestones

#### Integrations

- GitHub
- GitLab
- Slack
- Discord
- Email notifications

#### Productivity

- Saved templates
- Recurring issues
- Attachments
- Rich text editor
- Advanced search
- Shared saved views

#### Analytics

- Team reports
- Productivity insights
- Velocity tracking
- Burndown charts

#### Customization

- Custom workflows
- Custom issue types
- Custom fields
- Automation rules

### 9.3 Out of Scope

The following capabilities are intentionally excluded from Shipyard's MVP.

#### Enterprise Features

- Organization hierarchy
- Multiple organizations
- SSO
- SCIM provisioning
- Advanced audit logs
- Compliance management

#### Business Features

- CRM
- HR management
- Finance
- Accounting
- Billing management

#### AI Features

- AI issue generation
- AI sprint planning
- AI summaries
- AI assistants

#### Advanced Platform Features

- Plugin marketplace
- Public APIs
- Workflow automation engine
- Multi-region administration

### 9.4 Release Goals

Version 1.0 will be considered successful if users can:

- Create and manage workspaces.
- Collaborate with team members.
- Organize work using issues, projects, and cycles.
- Track progress from planning to completion.
- Receive notifications for Issue assignment or reassignment and comment mentions.
- Find work quickly through search and filters.
- Complete everyday project management tasks without requiring external tools.

### 9.5 Success Criteria

The MVP is complete when:

- All MVP functional requirements are implemented.
- Core user workflows are fully functional.
- Role-based permissions are enforced.
- Non-functional requirements are satisfied.
- The application is stable enough for daily use by small software teams.

## 10. Open Questions & Assumptions

### Overview

This section captures assumptions made during product planning and identifies questions that may require validation through user feedback, design exploration, or future product decisions.

### 10.1 Assumptions

The following assumptions guided the MVP definition:

#### Target Users

- Shipyard is designed primarily for software teams of 2-30 members.
- Teams prefer simplicity over extensive customization.
- Most users are familiar with modern project management tools.

#### Team Structure

- Every workspace has exactly one Owner.
- Teams have a relatively flat organizational structure.
- Projects are managed by Owners or Admins.
- Members primarily contribute by working on issues.

#### Workflow

- Teams organize work using Issues, Projects, and Cycles.
- Only one active cycle exists per workspace.
- Non-archived cycles never overlap within a workspace.
- Notifications help users stay informed but are not intended to replace communication tools.
- Search is a primary method of navigating large workspaces.

#### Product Philosophy

- Opinionated defaults are preferred over extensive configuration.
- The MVP prioritizes speed, clarity, and ease of use.
- Every feature should solve a common software development workflow.

### 10.2 Open Questions

The following decisions are intentionally deferred until future validation.

#### Workflow

- Should completed cycles generate automatic summary reports?
- Should users be allowed to watch issues they are not assigned to?

#### Collaboration

- Should emoji reactions be supported on comments?
- Should threaded discussions be introduced?
- Should users receive digest notifications in addition to real-time notifications?

#### Planning

- What is the best Kanban experience for Shipyard?
- Should Roadmaps become a core feature or remain optional?
- Should Milestones be separate entities or part of Projects?

#### Integrations

- Which third-party integration should be prioritized first?
- Should GitHub integration be one-way or two-way?
- Which notification channels should be supported beyond the in-app notification center?

### 10.3 Deferred Decisions

The following topics are intentionally postponed until after the MVP:

- Custom workflows.
- Custom issue types.
- Custom fields.
- Automation rules.
- API access.
- Plugin ecosystem.
- Organization support.
- Billing and subscriptions.
- AI-powered features.
- Advanced reporting and analytics.

### 10.4 Risks

Potential risks that may affect the product:

- Scope creep during MVP development.
- Feature parity expectations with established competitors.
- Performance challenges as workspace size grows.
- Balancing simplicity with flexibility.
- Prioritizing future features without direct user feedback.

### 10.5 Validation Plan

After the MVP is available, the team should gather feedback on:

- Overall usability.
- Issue management workflow.
- Project planning workflow.
- Cycle management.
- Notification usefulness.
- Search experience.
- Team collaboration.
- Missing features that impact daily workflows.
