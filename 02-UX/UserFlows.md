# User Flow 1 — User Registration

## Overview

This flow describes how a new user creates a Shipyard account. Registration is the first step in the onboarding experience and grants the user access to the platform.

---

## Trigger

The user selects **Sign Up** from the authentication screen.

---

## Preconditions

- The user does not have an existing Shipyard account.
- The user is not authenticated.

---

## Main Flow

1. The user opens the registration page.
2. The user enters:
   - Email address
   - Password
3. The user submits the registration form.
4. The system validates the submitted information.
5. The system creates the user account.
6. The user is automatically signed in.
7. The system redirects the user to the onboarding flow.
8. The user is prompted to create or join a workspace.

---

## Alternative Flows

### Email Already Exists

1. The user submits an email address that already exists.
2. The system displays an appropriate validation message.
3. The user remains on the registration page.

---

### Invalid Input

1. Required fields are missing or invalid.
2. Validation errors are displayed.
3. The user corrects the information.
4. Registration continues.

---

## Error Flows

### Server Error

1. Registration cannot be completed.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- A new Shipyard account exists.
- The user is authenticated.
- No workspace has been created yet.
- The onboarding process continues with workspace setup.

---

# User Flow 2 — User Login

## Overview

This flow describes how an existing user signs in to Shipyard and gains access to their workspaces.

---

## Trigger

The user selects **Log In** from the authentication screen.

---

## Preconditions

- The user has an existing Shipyard account.
- The user is not currently authenticated.

---

## Main Flow

1. The user opens the login page.
2. The user enters:
   - Email address
   - Password
3. The user submits the login form.
4. The system validates the credentials.
5. The user is successfully authenticated.
6. The system checks the user's workspaces.
7. If the user belongs to only one workspace, they are redirected to that workspace's dashboard.
8. If the user belongs to multiple workspaces, they are redirected to the workspace selection screen.
9. The user begins their session.

---

## Alternative Flows

### Single Workspace

1. The user belongs to only one workspace.
2. The system automatically opens that workspace.

---

### Multiple Workspaces

1. The user belongs to multiple workspaces.
2. The system displays the workspace selection screen.
3. The user selects a workspace.
4. The selected workspace is opened.

---

## Error Flows

### Invalid Credentials

1. The user enters an incorrect email or password.
2. The system displays a generic authentication error.
3. The user remains on the login page.

---

### Expired or Invalid Session

1. The user's session is no longer valid.
2. The system redirects the user to the login page.
3. The user signs in again.

---

### Server Error

1. Login cannot be completed.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- The user is authenticated.
- An active session is established.
- A workspace is selected.
- The user is redirected to the appropriate dashboard and can begin using Shipyard.

---

# User Flow 3 — First-Time Onboarding

## Overview

This flow describes the onboarding experience immediately after a user registers for Shipyard. The goal is to guide new users into a workspace as quickly as possible, either by creating a new workspace or joining an existing one through an invitation.

---

## Trigger

The user successfully completes account registration and signs in for the first time.

---

## Preconditions

- The user is authenticated.
- The user does not belong to any workspace.

---

## Main Flow

1. The system displays the onboarding screen.
2. The user is presented with two options:
   - Create a new workspace
   - Join an existing workspace
3. The user selects one of the options.

---

## Path A — Create First Workspace

1. The user selects **Create Workspace**.
2. The system displays the workspace creation form.
3. The user enters:
   - Workspace name
   - Workspace icon (optional)
4. The user submits the form.
5. The system creates the workspace.
6. The user becomes the Workspace Owner.
7. The system redirects the user to the new workspace dashboard.
8. The onboarding process is completed.

---

## Path B — Join Workspace via Invitation

1. The user selects **Join Workspace**.
2. The user enters or opens a valid invitation link.
3. The system validates the invitation.
4. The user accepts the invitation.
5. The system adds the user to the workspace.
6. The appropriate workspace role is assigned.
7. The system redirects the user to the workspace dashboard.
8. The onboarding process is completed.

---

## Alternative Flows

### Skip Workspace Creation

1. The user leaves the onboarding flow before creating or joining a workspace.
2. The next login returns the user to the onboarding screen until they join or create a workspace.

---

### Multiple Invitations

1. The user has multiple pending invitations.
2. The system displays all available invitations.
3. The user selects which workspace to join.

---

## Error Flows

### Invalid Invitation

1. The invitation link is invalid or expired.
2. The system displays an appropriate error message.
3. The user can enter another invitation or create a new workspace.

---

### Duplicate Workspace Name

1. The chosen workspace name is unavailable or invalid.
2. The system displays a validation message.
3. The user updates the workspace information.
4. Workspace creation continues.

---

### Server Error

1. The requested operation cannot be completed.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

### Create Workspace

- A new workspace exists.
- The user is the Workspace Owner.
- The workspace dashboard is displayed.

### Join Workspace

- The user becomes a member of the selected workspace.
- The assigned role is applied.
- The workspace dashboard is displayed.
- The onboarding process is complete.

---

# User Flow 4 — Enter Workspace

## Overview

This flow describes how an authenticated user enters a workspace after signing in. Depending on the number of workspaces the user belongs to, the system either opens the only available workspace automatically or prompts the user to select one.

---

## Trigger

The user successfully logs in to Shipyard.

---

## Preconditions

- The user is authenticated.
- The user belongs to at least one workspace.

---

## Main Flow

1. The system retrieves all workspaces associated with the user.
2. The system determines the number of available workspaces.
3. If only one workspace exists, it is opened automatically.
4. If multiple workspaces exist, the workspace selection screen is displayed.
5. The user selects a workspace.
6. The system loads the selected workspace.
7. The workspace dashboard is displayed.
8. The user's previous session context (if available) is restored.

---

## Alternative Flows

### Single Workspace

1. The user belongs to only one workspace.
2. The system skips the workspace selection screen.
3. The workspace dashboard opens immediately.

---

### Recently Used Workspace

1. The user belongs to multiple workspaces.
2. The system highlights the most recently used workspace.
3. The user can select it or choose another workspace.

---

## Error Flows

### Workspace No Longer Accessible

1. The selected workspace has been deleted or the user's access has been revoked.
2. The system removes the workspace from the available list.
3. The user selects another workspace.

---

### Failed Workspace Load

1. The workspace cannot be loaded.
2. The system displays an error message.
3. The user can retry loading the workspace or return to the workspace selection screen.

---

## Postconditions

- An active workspace is selected.
- Workspace-specific data is loaded.
- The dashboard is displayed.
- The user can begin interacting with projects, issues, cycles, and other workspace resources.

---
# User Flow 5 — Switch Workspace

## Overview

This flow describes how a user switches between workspaces during an active session. Workspace switching allows users who belong to multiple organizations or teams to quickly move between separate workspaces while maintaining a seamless experience.

---

## Trigger

The user selects the **Workspace Switcher** from the application interface.

---

## Preconditions

- The user is authenticated.
- The user belongs to two or more workspaces.
- A workspace is currently active.

---

## Main Flow

1. The user clicks the Workspace Switcher.
2. The system displays a list of workspaces the user has access to.
3. The currently active workspace is highlighted.
4. The user selects a different workspace.
5. The system verifies the user's access to the selected workspace.
6. The current workspace context is unloaded.
7. The selected workspace is loaded.
8. The user is redirected to the selected workspace's dashboard.
9. The Workspace Switcher updates to reflect the active workspace.

---

## Alternative Flows

### Return to Previously Used Workspace

1. The user opens the Workspace Switcher.
2. The user selects a previously used workspace.
3. The workspace loads successfully.
4. The dashboard for that workspace is displayed.

---

### Search for a Workspace

1. The user has access to many workspaces.
2. The user searches for a workspace by name.
3. The matching workspaces are displayed.
4. The user selects a workspace.
5. The selected workspace is opened.

---

## Error Flows

### Access Revoked

1. The user's access to the selected workspace has been removed.
2. The system displays an access error.
3. The unavailable workspace is removed from the list.
4. The user remains in the current workspace.

---

### Workspace Unavailable

1. The selected workspace cannot be loaded.
2. The system displays an error message.
3. The user can retry or remain in the current workspace.

---

## Postconditions

- The selected workspace becomes the active workspace.
- All workspace-specific data is refreshed.
- Navigation, projects, issues, cycles, members, and settings reflect the newly selected workspace.
- The user can continue working within the selected workspace.
---

# User Flow 6 — Invite Team Members

## Overview

This flow describes how a Workspace Owner or Administrator invites new members to join a workspace. Invitations allow teams to collaborate within the same workspace while maintaining appropriate access permissions.

---

## Trigger

The user selects **Invite Members** from the Members page or Workspace Settings.

---

## Preconditions

- The user is authenticated.
- A workspace is active.
- The user has permission to invite members.

---

## Main Flow

1. The user opens the **Members** page.
2. The user selects **Invite Members**.
3. The system displays the invitation dialog.
4. The user enters one or more email addresses.
5. The user selects the role for each invited member.
6. The user submits the invitation.
7. The system validates the entered email addresses.
8. The system generates invitation records.
9. Invitation emails are sent to the recipients.
10. The invited users appear in the Members list with a **Pending Invitation** status.
11. The inviter can continue working within the workspace.

---

## Alternative Flows

### Invite Multiple Members

1. The user enters multiple email addresses.
2. The system validates each address.
3. Invitations are created for all valid recipients.
4. Invalid recipients are identified individually.

---

### Resend Invitation

1. A pending invitation already exists.
2. The user selects **Resend Invitation**.
3. The system sends a new invitation email.
4. The invitation status remains **Pending**.

---

### Cancel Invitation

1. The user views a pending invitation.
2. The user selects **Cancel Invitation**.
3. The system invalidates the invitation.
4. The pending member is removed from the invitation list.

---

## Error Flows

### Invalid Email Address

1. One or more email addresses are invalid.
2. The system highlights the invalid entries.
3. The user corrects the information.
4. The invitation process continues.

---

### User Already a Member

1. The entered email belongs to an existing workspace member.
2. The system informs the inviter that the user is already part of the workspace.
3. No duplicate invitation is created.

---

### Existing Pending Invitation

1. A pending invitation already exists for the entered email.
2. The system informs the inviter.
3. The inviter may resend or cancel the existing invitation.

---

### Insufficient Permissions

1. The user attempts to invite members without the required permission.
2. The system denies the action.
3. An appropriate error message is displayed.

---

### Server Error

1. The invitation cannot be processed.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- Valid invitations have been created.
- Invitation emails have been sent.
- Pending invitations are visible in the Members page.
- Invited users can join the workspace using their invitation.
---
# User Flow 7 — Accept Workspace Invitation

## Overview

This flow describes how a user accepts an invitation to join an existing workspace. The invitation may be accepted by an existing Shipyard user or by a new user who must first create an account before joining.

---

## Trigger

The user opens a valid workspace invitation link received via email.

---

## Preconditions

- A valid invitation exists.
- The invitation has not expired or been revoked.
- The workspace still exists.

---

## Main Flow

1. The user opens the invitation link.
2. The system validates the invitation.
3. If the user is not authenticated, they are prompted to log in or create an account.
4. After authentication, the invitation details are displayed, including:
   - Workspace name
   - Invited role
   - Inviter (optional)
5. The user selects **Accept Invitation**.
6. The system verifies that the invitation is still valid.
7. The system adds the user to the workspace.
8. The assigned role is applied.
9. The invitation is marked as accepted.
10. The system redirects the user to the workspace dashboard.
11. The user becomes an active workspace member.

---

## Alternative Flows

### Existing Shipyard User

1. The user is already signed in.
2. The system skips authentication.
3. The invitation details are displayed.
4. The user accepts the invitation.
5. The workspace is added to the user's workspace list.

---

### New Shipyard User

1. The user does not have a Shipyard account.
2. The system redirects the user to the registration page.
3. The user creates an account.
4. Authentication is completed.
5. The invitation is automatically resumed.
6. The user accepts the invitation.
7. The workspace is opened.

---

### User Already a Member

1. The invited user is already a member of the workspace.
2. The system informs the user.
3. The workspace is opened directly.

---

## Error Flows

### Invitation Expired

1. The invitation has expired.
2. The system informs the user.
3. The user is instructed to request a new invitation.

---

### Invitation Revoked

1. The invitation has been cancelled by the workspace administrator.
2. The system displays an appropriate message.
3. The invitation cannot be accepted.

---

### Workspace No Longer Exists

1. The invitation references a deleted workspace.
2. The system displays an error.
3. The invitation becomes invalid.

---

### Authentication Failure

1. The user cannot complete authentication.
2. The invitation remains pending.
3. The user can retry authentication later.

---

### Server Error

1. The invitation cannot be processed.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- The user becomes an active member of the workspace.
- The assigned workspace role is applied.
- The invitation is marked as accepted and can no longer be reused.
- The workspace is added to the user's workspace list.
- The workspace dashboard is displayed, allowing the user to begin collaborating immediately.
---
# User Flow 8 — Manage Members & Roles

## Overview

This flow describes how Workspace Owners and Administrators manage workspace members after they have joined. Member management includes viewing members, updating roles, removing members, and maintaining appropriate access permissions.

---

## Trigger

The user opens the **Members** page or **Member Management** section in Workspace Settings.

---

## Preconditions

- The user is authenticated.
- A workspace is active.
- The user has permission to manage members.
- At least one member exists in the workspace.

---

## Main Flow

### View Members

1. The user opens the **Members** page.
2. The system displays all workspace members.
3. Each member includes:
   - Name
   - Email
   - Role
   - Status
   - Join Date
4. The user selects a member to manage.

---

### Change Member Role

1. The user selects a member.
2. The user chooses **Change Role**.
3. The system displays the available roles.
4. The user selects a new role.
5. The system validates the permission change.
6. The member's role is updated.
7. The Members list reflects the new role immediately.

---

### Remove Member

1. The user selects a member.
2. The user chooses **Remove Member**.
3. The system displays a confirmation dialog.
4. The user confirms the action.
5. The system removes the member from the workspace.
6. The removed user immediately loses access to the workspace.
7. The Members list is updated.

---

## Alternative Flows

### View Pending Invitations

1. The user switches to the **Pending Invitations** section.
2. The system displays all pending invitations.
3. The user can resend or cancel invitations.

---

### Cancel Pending Invitation

1. The user selects a pending invitation.
2. The user chooses **Cancel Invitation**.
3. The system removes the pending invitation.
4. The invitee can no longer use the invitation link.

---

### Resend Invitation

1. The user selects a pending invitation.
2. The user chooses **Resend Invitation**.
3. The system sends a new invitation email.
4. The invitation remains pending.

---

## Error Flows

### Insufficient Permissions

1. A user without the required permission attempts to manage members.
2. The system blocks the action.
3. An appropriate error message is displayed.

---

### Cannot Modify Workspace Owner

1. A user attempts to change or remove the Workspace Owner.
2. The system prevents the action.
3. An explanatory message is displayed.

---

### Attempt to Remove Yourself

1. The user attempts to remove themselves from the workspace.
2. The system prevents the action if it would leave the workspace without an owner.
3. Otherwise, the user is asked to confirm leaving the workspace.

---

### Server Error

1. The requested operation cannot be completed.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- Member information reflects the latest changes.
- Updated roles take effect immediately.
- Removed members no longer have access to the workspace.
- Pending invitations accurately reflect their current status.
- Workspace permissions remain consistent with the assigned roles.
---
# User Flow 9 — Create Project

## Overview

This flow describes how a workspace member with the appropriate permissions creates a new project. A project groups related issues and provides a central place to plan, organize, and monitor work toward a common objective.

---

## Trigger

The user selects **Create Project** from the Projects page or the global **Create** menu.

---

## Preconditions

- The user is authenticated.
- A workspace is active.
- The user has permission to create projects.

---

## Main Flow

1. The user navigates to the **Projects** page.
2. The user selects **Create Project**.
3. The system displays the project creation form.
4. The user enters the project details:
   - Project name
   - Description (optional)
   - Project icon (optional)
   - Start date (optional)
   - Target date (optional)
5. The user submits the form.
6. The system validates the provided information.
7. The project is created.
8. The project is added to the workspace's Projects list.
9. The user is redirected to the project's overview page.
10. The project is ready to receive issues.

---

## Alternative Flows

### Create Project from Global Menu

1. The user selects the global **Create** button.
2. The user chooses **Project**.
3. The project creation form is displayed.
4. The remaining steps follow the main flow.

---

### Create Project Without Optional Fields

1. The user enters only the required information.
2. The system creates the project successfully.
3. Optional details can be added later.

---

## Error Flows

### Missing Required Information

1. The user submits the form without the required fields.
2. The system highlights the missing or invalid information.
3. The user corrects the form.
4. Project creation continues.

---

### Duplicate Project Name

1. A project with the same name already exists within the workspace.
2. The system informs the user.
3. The user chooses a different name or confirms creation if duplicate names are permitted.

---

### Insufficient Permissions

1. The user attempts to create a project without the required permission.
2. The system blocks the action.
3. An appropriate error message is displayed.

---

### Server Error

1. The project cannot be created.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- A new project exists within the workspace.
- The project appears in the Projects list.
- The project overview page is displayed.
- The project is ready for cycles and issues to be associated with it.

---
# User Flow 10 — Manage Project

## Overview

This flow describes how users manage a project's lifecycle after it has been created. Project management includes viewing project details, updating project information, monitoring progress, changing project status, archiving completed projects, and deleting projects when appropriate.

---

## Trigger

The user opens an existing project from the Projects page.

---

## Preconditions

- The user is authenticated.
- A workspace is active.
- At least one project exists.
- The user has permission to manage the project.

---

## Main Flow

### View Project

1. The user navigates to the **Projects** page.
2. The user selects a project.
3. The system displays the project overview, including:
   - Project name
   - Description
   - Current status
   - Progress
   - Associated issues
   - Assigned cycle(s)
   - Team members
   - Activity history

---

### Edit Project

1. The user selects **Edit Project**.
2. The system displays the editable project information.
3. The user updates one or more project details.
4. The user saves the changes.
5. The system validates and updates the project.
6. The updated information is displayed immediately.

---

### Update Project Status

1. The user changes the project status.
2. The system updates the project.
3. The new status is reflected throughout the workspace.

---

### Archive Project

1. The user selects **Archive Project**.
2. The system displays a confirmation dialog.
3. The user confirms the action.
4. The project is moved to the Archived Projects list.
5. The project becomes read-only but remains accessible.

---

### Restore Project

1. The user opens the Archived Projects list.
2. The user selects **Restore Project**.
3. The project returns to the active Projects list.
4. The project becomes editable again.

---

### Delete Project

1. The user selects **Delete Project**.
2. The system displays a confirmation dialog explaining the consequences.
3. The user confirms the deletion.
4. The project is permanently removed from the workspace.

---

## Alternative Flows

### View Archived Projects

1. The user navigates to the Archived Projects section.
2. The system displays archived projects.
3. The user can inspect or restore a project.

---

### Read-Only Access

1. A user without edit permissions opens a project.
2. The project details are displayed.
3. Editing actions are hidden or disabled.

---

## Error Flows

### Insufficient Permissions

1. The user attempts a management action without the required permission.
2. The system blocks the action.
3. An appropriate error message is displayed.

---

### Active Dependencies

1. The user attempts to delete a project that still contains active issues.
2. The system prevents deletion.
3. The user is instructed to resolve, move, or archive the remaining issues before deleting the project.

---

### Server Error

1. The requested operation cannot be completed.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- The project reflects the latest changes.
- Status updates are visible throughout the workspace.
- Archived projects remain available for future restoration.
- Deleted projects are removed according to workspace policies.
- All project information remains consistent across related issues and cycles.

---
# User Flow 11 — Create Cycle

## Overview

This flow describes how a workspace member with the appropriate permissions creates a new cycle. A cycle represents a fixed period of work used to organize and track issues toward a short-term goal.

---

## Trigger

The user selects **Create Cycle** from the Cycles page or the global **Create** menu.

---

## Preconditions

- The user is authenticated.
- A workspace is active.
- The user has permission to create cycles.

---

## Main Flow

1. The user navigates to the **Cycles** page.
2. The user selects **Create Cycle**.
3. The system displays the cycle creation form.
4. The user enters the cycle details:
   - Cycle name
   - Description (optional)
   - Start date
   - End date
5. The user submits the form.
6. The system validates the provided information.
7. The cycle is created.
8. The cycle is added to the workspace's Cycles list.
9. The user is redirected to the cycle overview page.
10. The cycle is ready to receive issues.

---

## Alternative Flows

### Create Cycle from Global Menu

1. The user selects the global **Create** button.
2. The user chooses **Cycle**.
3. The cycle creation form is displayed.
4. The remaining steps follow the main flow.

---

### Create Cycle Without Description

1. The user provides only the required information.
2. The system creates the cycle successfully.
3. The description can be added later.

---

## Error Flows

### Missing Required Information

1. The user submits the form without all required fields.
2. The system highlights the missing or invalid information.
3. The user corrects the form.
4. Cycle creation continues.

---

### Invalid Date Range

1. The selected end date occurs before the start date.
2. The system displays a validation message.
3. The user selects a valid date range.
4. Cycle creation continues.

---

### Overlapping Cycle

1. The selected date range overlaps an existing active cycle.
2. The system warns the user according to workspace rules.
3. The user can adjust the dates or continue if overlapping cycles are permitted.

---

### Insufficient Permissions

1. The user attempts to create a cycle without the required permission.
2. The system blocks the action.
3. An appropriate error message is displayed.

---

### Server Error

1. The cycle cannot be created.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- A new cycle exists within the workspace.
- The cycle appears in the Cycles list.
- The cycle overview page is displayed.
- The cycle is ready for issues to be planned and tracked.

---

# User Flow 12 — Manage Cycle

## Overview

This flow describes how users manage a cycle after it has been created. Cycle management includes viewing cycle progress, updating cycle details, monitoring completion, closing completed cycles, reopening cycles when necessary, and archiving historical cycles.

---

## Trigger

The user opens an existing cycle from the Cycles page.

---

## Preconditions

- The user is authenticated.
- A workspace is active.
- At least one cycle exists.
- The user has permission to manage cycles.

---

## Main Flow

### View Cycle

1. The user navigates to the **Cycles** page.
2. The user selects a cycle.
3. The system displays the cycle overview, including:
   - Cycle name
   - Description
   - Start date
   - End date
   - Current status
   - Progress
   - Associated issues
   - Completion statistics
   - Activity history

---

### Edit Cycle

1. The user selects **Edit Cycle**.
2. The system displays the editable cycle information.
3. The user updates one or more cycle details.
4. The user saves the changes.
5. The system validates the updated information.
6. The cycle information is updated immediately.

---

### Update Cycle Status

1. The user changes the cycle status.
2. The system updates the cycle.
3. The new status is reflected throughout the workspace.

---

### Complete Cycle

1. The user selects **Complete Cycle**.
2. The system evaluates all issues assigned to the cycle.
3. The user reviews any incomplete issues.
4. The user chooses how incomplete issues should be handled (for example, moving them to another cycle or leaving them in the backlog).
5. The user confirms completion.
6. The system closes the cycle.
7. The cycle becomes read-only.

---

### Reopen Cycle

1. The user opens a completed cycle.
2. The user selects **Reopen Cycle**.
3. The system restores the cycle to an active state.
4. Issues can once again be updated within the cycle.

---

### Archive Cycle

1. The user selects **Archive Cycle**.
2. The system displays a confirmation dialog.
3. The user confirms the action.
4. The cycle is moved to the Archived Cycles list.
5. The archived cycle remains available for historical reference.

---

## Alternative Flows

### View Archived Cycles

1. The user opens the Archived Cycles section.
2. The system displays archived cycles.
3. The user can inspect or restore archived cycles.

---

### Read-Only Access

1. A user without edit permissions opens a cycle.
2. The cycle information is displayed.
3. Management actions are hidden or disabled.

---

## Error Flows

### Insufficient Permissions

1. The user attempts to manage a cycle without the required permission.
2. The system blocks the action.
3. An appropriate error message is displayed.

---

### Invalid Date Update

1. The user modifies the cycle dates to an invalid range.
2. The system displays a validation message.
3. The user corrects the dates.
4. The update continues.

---

### Server Error

1. The requested operation cannot be completed.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- The cycle reflects the latest updates.
- Progress and completion statistics are updated.
- Completed cycles preserve their historical information.
- Archived cycles remain available for future reference.
- Related issues remain consistent with the updated cycle state.

---

# User Flow 13 — Create Issue

## Overview

This flow describes how a workspace member creates a new issue. An issue represents a unit of work that can be planned, assigned, prioritized, tracked, and completed throughout its lifecycle.

---

## Trigger

The user selects **Create Issue** from the Issues page, a Project, a Cycle, or the global **Create** menu.

---

## Preconditions

- The user is authenticated.
- A workspace is active.
- The user has permission to create issues.

---

## Main Flow

1. The user initiates the **Create Issue** action.
2. The system displays the issue creation form.
3. The user enters the issue details, including:
   - Title
   - Description (optional)
   - Project (optional)
   - Cycle (optional)
   - Assignee (optional)
   - Priority
   - Labels (optional)
   - Due Date (optional)
4. The user submits the form.
5. The system validates the provided information.
6. The issue is created.
7. The system generates a unique issue identifier.
8. The issue is added to the appropriate issue list.
9. If a Project or Cycle was selected, the issue is automatically associated with it.
10. The issue details page is displayed.

---

## Alternative Flows

### Quick Create

1. The user opens the global **Create** menu.
2. The user enters only:
   - Title
   - Priority (optional)
3. The system creates the issue immediately.
4. Remaining information can be added later.

---

### Create from Project

1. The user opens a project.
2. The user selects **Create Issue**.
3. The selected project is automatically assigned.
4. The remaining steps follow the main flow.

---

### Create from Cycle

1. The user opens a cycle.
2. The user selects **Create Issue**.
3. The selected cycle is automatically assigned.
4. The remaining steps follow the main flow.

---

### Create from Backlog

1. The user creates an issue from the Backlog view.
2. No project or cycle is assigned initially.
3. The issue remains in the backlog until planned.

---

## Error Flows

### Missing Required Information

1. The user submits the form without a title.
2. The system highlights the missing field.
3. The user provides the required information.
4. Issue creation continues.

---

### Invalid Assignment

1. The selected project, cycle, or assignee is no longer available.
2. The system notifies the user.
3. The user selects another option.
4. Issue creation continues.

---

### Insufficient Permissions

1. The user attempts to create an issue without the required permission.
2. The system blocks the action.
3. An appropriate error message is displayed.

---

### Server Error

1. The issue cannot be created.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- A new issue exists within the workspace.
- The issue receives a unique identifier.
- Any selected Project, Cycle, and Assignee are linked to the issue.
- The issue is visible in the relevant issue views.
- The issue is ready to progress through its workflow.

---
# User Flow 14 — Manage Issue

## Overview

This flow describes how users manage an issue after it has been created. Issue management includes viewing issue details, updating planning information, assigning ownership, organizing the issue, and archiving or deleting it when appropriate.

---

## Trigger

The user opens an existing issue from the Dashboard, Issues page, Project, Cycle, Search results, or a Notification.

---

## Preconditions

- The user is authenticated.
- A workspace is active.
- The issue exists.
- The user has permission to view the issue.

---

## Main Flow

### View Issue

1. The user opens an issue.
2. The system displays the issue details, including:
   - Issue Identifier
   - Title
   - Description
   - Status
   - Priority
   - Assignee
   - Project
   - Cycle
   - Labels
   - Due Date
   - Creator
   - Created Date
   - Last Updated
   - Activity History

---

### Edit Issue Details

1. The user selects **Edit**.
2. The user updates one or more issue fields.
3. The system validates the changes.
4. The issue is updated immediately.

---

### Assign or Reassign Issue

1. The user selects the **Assignee** field.
2. The system displays eligible workspace members.
3. The user selects a member or clears the assignment.
4. The issue is updated.
5. If assigned or reassigned, the assignee receives a notification.

---

### Update Planning Information

The user may update:

- Priority
- Labels
- Project
- Cycle
- Due Date

The system validates each change and immediately updates the issue.

---

### Archive Issue

1. The user selects **Archive Issue**.
2. The system displays a confirmation dialog.
3. The user confirms the action.
4. The issue becomes read-only.
5. The issue is removed from active issue lists.

---

### Delete Issue

1. The user selects **Delete Issue**.
2. The system displays a confirmation dialog.
3. The user confirms the deletion.
4. The issue is permanently removed.

---

## Alternative Flows

### View Issue History

1. The user opens the Activity History section.
2. The system displays chronological updates, including:
   - Field changes
   - Status changes
   - Assignment changes
   - Creation information

---

### Read-Only Issue

1. The user opens an archived issue.
2. The issue information is displayed.
3. Editing actions are disabled.

---

## Error Flows

### Issue No Longer Exists

1. The issue has been deleted.
2. The system informs the user.
3. The user is redirected to the Issues page.

---

### Invalid Assignment

1. The selected assignee is no longer available.
2. The system notifies the user.
3. The user selects another assignee.

---

### Insufficient Permissions

1. The user attempts a restricted action.
2. The system blocks the action.
3. An appropriate error message is displayed.

---

### Simultaneous Updates

1. Another user updates the issue at the same time.
2. The system refreshes the issue with the latest information.
3. The user is informed if their changes cannot be applied.

---

### Server Error

1. The requested operation cannot be completed.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- The issue reflects the latest information.
- Assignment and planning changes are immediately visible throughout the workspace.
- Archived issues remain searchable but cannot be modified.
- Deleted issues are permanently removed according to workspace policies.
- All changes are recorded in the issue activity history.
---
# User Flow 15 — Issue Workflow

## Overview

This flow describes how an issue progresses through its lifecycle. It covers status transitions from creation to completion, ensuring work is tracked consistently while maintaining a complete history of changes.

---

## Trigger

A user updates an issue's status from the issue details page or an issue list.

---

## Preconditions

- The user is authenticated.
- A workspace is active.
- The issue exists.
- The user has permission to update the issue.

---

## Main Flow

### Move Issue Through Workflow

1. A new issue is created.
2. The issue is automatically assigned the **Backlog** status.
3. The user opens the issue or updates its status.
4. The user moves the issue through the workflow:
   - Backlog
   - Todo
   - In Progress
   - Done
5. The system updates the issue status.
6. The status change is immediately reflected throughout the workspace.
7. The system records the status change in the issue activity history.

---

### Mark Work as Started

1. The user changes the issue status from **Todo** to **In Progress**.
2. The system updates the issue.
3. Team members can immediately see that work has started.

---

### Complete Work

1. The user changes the issue status to **Done**.
2. The system marks the issue as completed.
3. The completed issue remains searchable and visible in its associated Project and Cycle.

---

## Alternative Flows

### Return Work to Todo

1. The user determines that work cannot continue.
2. The issue status is changed from **In Progress** back to **Todo**.
3. The workflow continues from the updated status.

---

### Return Completed Work

1. A completed issue requires additional work.
2. The user changes the status from **Done** to **In Progress** or **Todo**.
3. The issue becomes active again.
4. The activity history records the status change.

---

### Move Directly from Backlog

1. The user prioritizes newly created work.
2. The issue is moved directly from **Backlog** to **In Progress** or **Done**, if appropriate.
3. The workflow history records the transition.

---

## Error Flows

### Archived Issue

1. The user attempts to change the status of an archived issue.
2. The system blocks the action.
3. The issue remains read-only.

---

### Insufficient Permissions

1. The user attempts to update an issue without sufficient permissions.
2. The system rejects the action.
3. An appropriate error message is displayed.

---

### Simultaneous Status Update

1. Another user changes the issue status at the same time.
2. The system refreshes the issue with the latest information.
3. The user is informed if their update cannot be applied.

---

### Server Error

1. The status update cannot be completed.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- The issue reflects its current workflow status.
- Project and Cycle progress update automatically when applicable.
- The issue activity history records every status transition.
- Completed issues remain searchable.
- Archived issues cannot participate in the workflow.
---
# User Flow 16 — Comments & Mentions

## Overview

This flow describes how workspace members collaborate on an issue by adding comments, editing or deleting their own comments, and mentioning teammates. Comments provide contextual discussions, while mentions notify relevant members about work that requires their attention.

---

## Trigger

The user opens an active issue and interacts with the Comments section.

---

## Preconditions

- The user is authenticated.
- A workspace is active.
- The issue exists.
- The issue is not archived.
- The user has permission to comment.

---

## Main Flow

### Add Comment

1. The user opens an issue.
2. The user navigates to the **Comments** section.
3. The user enters a comment.
4. The user submits the comment.
5. The system validates the input.
6. The comment is added to the issue.
7. The comment appears in chronological order.

---

### Mention a Team Member

1. While writing a comment, the user types **@**.
2. The system displays matching workspace members.
3. The user selects a member.
4. The mention is inserted into the comment.
5. The comment is submitted.
6. The mentioned user receives a notification.

---

### Edit Comment

1. The comment author selects **Edit**.
2. The user updates the comment.
3. The user saves the changes.
4. The system updates the comment.

---

### Delete Comment

1. The comment author selects **Delete**.
2. The system displays a confirmation dialog.
3. The user confirms the action.
4. The comment is removed from the issue.

---

## Alternative Flows

### Moderator Removes Comment

1. A user with elevated permissions selects a comment.
2. The user removes the comment.
3. The comment is deleted from the issue.

---

### Multiple Mentions

1. The user mentions multiple workspace members within a comment.
2. The comment is submitted.
3. Each mentioned member receives an individual notification.

---

## Error Flows

### Empty Comment

1. The user submits an empty comment.
2. The system rejects the submission.
3. The user is prompted to enter content.

---

### Mention Invalid User

1. The user attempts to mention a user who does not exist or is no longer a workspace member.
2. The system prevents the invalid mention.
3. The user selects another member or removes the mention.

---

### Archived Issue

1. The user attempts to add or edit a comment on an archived issue.
2. The system blocks the action.
3. Existing comments remain visible, but no new comments can be added or modified.

---

### Insufficient Permissions

1. The user attempts to remove another user's comment without sufficient permissions.
2. The system rejects the action.
3. An appropriate error message is displayed.

---

### Server Error

1. The requested operation cannot be completed.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- Comments remain associated with the issue.
- Comments are displayed in chronological order.
- Mentioned users receive notifications.
- Comment edits and deletions are reflected immediately.
- Archived issues preserve their discussion history but cannot receive new comments.
---
# User Flow 17 — Notifications

## Overview

This flow describes how users review and manage notifications within their workspace. Notifications keep users informed about relevant events and provide quick access to related resources.

---

## Trigger

The user opens the Notifications panel.

---

## Preconditions

- The user is authenticated.
- A workspace is active.
- The user has one or more notifications.

---

## Main Flow

### View Notifications

1. The user opens the **Notifications** panel.
2. The system displays notifications in reverse chronological order.
3. Each notification displays:
   - Notification type
   - Message
   - Related item (when applicable)
   - Timestamp
   - Read status

---

### Open Notification

1. The user selects a notification.
2. The system marks the notification as read.
3. If the notification references a resource, the user is navigated to the related item.

---

### Mark Notification as Read

1. The user selects **Mark as Read**.
2. The system updates the notification status.
3. The notification remains available in the notification list.

---

### Mark All as Read

1. The user selects **Mark All as Read**.
2. The system updates all unread notifications.
3. All notifications are now marked as read.

---

### Delete Notification

1. The user selects **Delete**.
2. The notification is removed from the notification list.

---

### Clear All Notifications

1. The user selects **Clear All**.
2. The system displays a confirmation dialog.
3. The user confirms the action.
4. All notifications are removed.

---

## Alternative Flows

### View Only Unread Notifications

1. The user filters the notification list.
2. The system displays only unread notifications.

---

### Review Previously Read Notifications

1. The user views the complete notification history.
2. Read notifications remain accessible until deleted.

---

## Error Flows

### Referenced Resource No Longer Exists

1. The user opens a notification.
2. The related issue, project, or cycle has been deleted.
3. The system informs the user that the resource is no longer available.

---

### Notification References Archived Resource

1. The user opens a notification.
2. The related resource has been archived.
3. The system opens the archived resource in read-only mode.

---

### Server Error

1. A notification action cannot be completed.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- Notification read status reflects the latest user actions.
- Deleted notifications are permanently removed.
- Remaining notifications continue to provide quick access to related resources.
- Users stay informed about relevant workspace activity.
---
# User Flow 18 — Dashboard

## Overview

This flow describes how users use the Dashboard as the primary entry point into their workspace. The Dashboard provides a personalized overview of work, recent activity, and quick access to common actions.

---

## Trigger

The user enters a workspace or navigates to the Dashboard.

---

## Preconditions

- The user is authenticated.
- A workspace is active.

---

## Main Flow

### View Dashboard

1. The user opens the Dashboard.
2. The system displays a personalized workspace overview, including:
   - Assigned Issues
   - Created Issues
   - Recently Viewed Issues
   - Overdue Issues
   - Active Projects
   - Current Active Cycle
   - Recent Workspace Activity
   - Recently Completed Issues
   - Quick Actions

---

### Resume Work

1. The user reviews assigned or recently viewed issues.
2. The user selects an issue.
3. The system opens the issue details.

---

### Monitor Team Progress

1. The user reviews active projects.
2. The user reviews the current active cycle.
3. The user selects a project or cycle.
4. The system navigates to the selected resource.

---

### Use Quick Actions

1. The user selects a quick action.
2. Available actions include:
   - Create Issue
   - Search Workspace
   - View Assigned Issues
3. The system navigates to the selected feature.

---

### Review Recent Activity

1. The user views the recent workspace activity feed.
2. The user selects an activity item.
3. The system opens the related resource.

---

## Alternative Flows

### No Assigned Issues

1. The user has no assigned issues.
2. The Dashboard displays an appropriate empty state.

---

### No Active Cycle

1. The workspace has no active cycle.
2. The Dashboard indicates that no active cycle exists.

---

### No Recent Activity

1. The workspace has no recent activity.
2. The Dashboard displays an appropriate empty state.

---

## Error Flows

### Dashboard Data Cannot Be Loaded

1. The Dashboard fails to load.
2. The system displays an error message.
3. The user can retry loading the Dashboard.

---

### Partial Data Unavailable

1. One or more Dashboard sections cannot be loaded.
2. Available sections continue to function.
3. Unavailable sections display an appropriate error or placeholder.

---

## Postconditions

- The user has an up-to-date overview of their workspace.
- The user can quickly access relevant work.
- The Dashboard reflects the latest workspace information.
- The user can continue to other product workflows through the available navigation and quick actions.
---
# User Flow 19 — Search & Filters

## Overview

This flow describes how users search for and organize workspace resources. Search and filters help users quickly locate issues, projects, cycles, and members while supporting efficient navigation in larger workspaces.

---

## Trigger

The user opens the Search interface or applies filters within a supported resource page.

---

## Preconditions

- The user is authenticated.
- A workspace is active.

---

## Main Flow

### Search Workspace

1. The user opens the Search interface.
2. The user enters a search query.
3. The system searches the current workspace.
4. Matching resources are displayed, including:
   - Issues
   - Projects
   - Cycles
   - Members
5. The user selects a result.
6. The system opens the selected resource.

---

### Filter Results

1. The user opens the filter options.
2. The user applies one or more filters, including:
   - Status
   - Priority
   - Assignee
   - Project
   - Cycle
   - Labels
   - Creator
   - Due Date
3. The system updates the results immediately.

---

### Sort Results

1. The user selects a sorting option.
2. Available options include:
   - Created Date
   - Updated Date
   - Due Date
   - Priority
   - Title
3. The system reorders the results.

---

### Save Filter Configuration

1. The user configures search filters.
2. The user selects **Save View**.
3. The user provides a name.
4. The system saves the filter configuration as a private view.

---

### Manage Saved Views

1. The user opens Saved Views.
2. The user may:
   - Open a saved view
   - Rename a saved view
   - Delete a saved view
   - Set a saved view as the default
3. The selected action is applied immediately.

---

## Alternative Flows

### Empty Search Query

1. The user opens Search without entering a query.
2. The system displays an empty search state.

---

### No Matching Results

1. The search or filters return no matches.
2. The system displays an appropriate empty state.
3. The user may refine or clear the search criteria.

---

### Multiple Filters

1. The user applies multiple filters simultaneously.
2. The system combines all filters.
3. Only matching resources are displayed.

---

## Error Flows

### Invalid Filter Combination

1. The selected filters produce no valid results.
2. The system displays an empty state.
3. The user may adjust the filters.

---

### Saved View References Deleted Resource

1. The user opens a saved view.
2. One or more referenced resources no longer exist.
3. The system applies the remaining valid filters.
4. Invalid references are ignored.

---

### Server Error

1. Search results cannot be loaded.
2. The system displays an error message.
3. The user can retry the search.

---

## Postconditions

- Matching resources are displayed.
- Applied filters and sorting are reflected immediately.
- Private saved views are available for future use.
- Users can quickly navigate to relevant workspace resources.
---
# User Flow 20 — Settings

## Overview

This flow describes how users manage their personal account settings and workspace configuration. It includes updating profile information, managing workspace settings, and ending an active session.

---

## Trigger

The user opens the **Settings** page from the workspace navigation or user menu.

---

## Preconditions

- The user is authenticated.
- A workspace is active.

---

## Main Flow

### Manage Profile

1. The user opens **Profile Settings**.
2. The system displays the user's profile information.
3. The user updates one or more fields, including:
   - Full Name
   - Profile Picture
4. The user saves the changes.
5. The system validates the input.
6. The updated profile information is saved and reflected throughout the workspace.

---

### Change Password

1. The user opens **Security Settings**.
2. The user enters:
   - Current Password
   - New Password
   - Confirm New Password
3. The user submits the changes.
4. The system validates the credentials.
5. The password is updated successfully.

---

### Manage Workspace Settings

1. A user with the required permissions opens **Workspace Settings**.
2. The system displays the current workspace configuration.
3. The user updates one or more settings, including:
   - Workspace Name
   - Workspace Logo
4. The user saves the changes.
5. The system validates the input.
6. The workspace settings are updated.

---

### Logout

1. The user opens the account menu.
2. The user selects **Logout**.
3. The system ends the active session.
4. The user is redirected to the login page.

---

## Alternative Flows

### Cancel Changes

1. The user modifies one or more settings.
2. The user cancels the operation.
3. No changes are saved.

---

### Read-Only Workspace Settings

1. A workspace member without administrative permissions opens Workspace Settings.
2. The system prevents editing of workspace configuration.

---

## Error Flows

### Invalid Profile Information

1. The submitted profile information fails validation.
2. The system displays the validation errors.
3. The user corrects the input.

---

### Incorrect Current Password

1. The user enters an incorrect current password.
2. The system rejects the password change.
3. The user is prompted to try again.

---

### Insufficient Permissions

1. The user attempts to modify workspace settings without sufficient permissions.
2. The system rejects the action.
3. An appropriate error message is displayed.

---

### Server Error

1. A settings update cannot be completed.
2. The system displays an error message.
3. The user can retry the operation.

---

## Postconditions

- Updated profile information is reflected throughout the workspace.
- Workspace settings reflect the latest authorized changes.
- Password changes take effect immediately.
- Logout terminates the active session and returns the user to the login page.
----
