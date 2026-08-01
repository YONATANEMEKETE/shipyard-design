Empty-state actions are filtered using the current user's permissions. Unauthorized actions are omitted rather than displayed in a disabled state.

## 6.2 Workspace

| Screen / Section | Condition | Message | Primary Action | Secondary Action |
|------------------|-----------|---------|----------------|------------------|
| Workspace Onboarding | User has no workspace | Welcome! Create your first workspace to get started. | Create Workspace | — |
| Workspace Switcher | User belongs to only one workspace | No additional workspaces available. | Create Workspace | — |
| Archived Workspaces | Workspace Owner has no archived workspaces | No archived workspaces available. | Back to Workspaces | — |
| Workspace Members | Workspace has no members other than the owner | You're the only member in this workspace. Invite teammates to start collaborating. | Invite Members | — |

---
## 6.3 Members

| Screen / Section | Condition | Message | Primary Action | Secondary Action |
|------------------|-----------|---------|----------------|------------------|
| Members Directory | No members match the current search or filters | No matching members found. Try adjusting your search or filters. | Clear Filters | Invite Members *(Owner/Admin only)* |

---
## 6.4 Projects

| Screen / Section | Condition | Message | Primary Action | Secondary Action |
|------------------|-----------|---------|----------------|------------------|
| Projects List *(Owner/Admin)* | No projects exist in the workspace | No projects yet. Create your first project to start organizing work. | Create Project | — |
| Projects List *(Member)* | No projects exist in the workspace | No projects yet. Projects created by a workspace Owner or Admin will appear here. | — | — |
| Projects List | No projects match the current search or filters | No matching projects found. Try adjusting your search or filters. | Clear Filters | Create Project *(Owner/Admin only)* |
| Project Kanban Column | No matching Projects have the column's status | No projects in this status. Authorized editors can drag a project here to update its status. | — | — |
| Archived Projects | No archived projects exist | No archived projects available. | View Active Projects | — |
| Project Details → Issues | The project has no issues | This project doesn't have any issues yet. Create the first issue to start tracking work. | Create Issue | — |
| Project Activity | No project activity has been recorded | No activity yet. Project updates will appear here as work progresses. | — | — |

---
## 6.5 Cycles

| Screen / Section | Condition | Message | Primary Action | Secondary Action |
|------------------|-----------|---------|----------------|------------------|
| Cycles List *(Owner/Admin)* | No cycles exist in the workspace | No cycles yet. Create your first cycle to start planning work. | Create Cycle | — |
| Cycles List *(Member)* | No cycles exist in the workspace | No cycles yet. Cycles created by a workspace Owner or Admin will appear here. | — | — |
| Cycles List | No cycles match the current search or filters | No matching cycles found. Try adjusting your search or filters. | Clear Filters | Create Cycle *(Owner/Admin only)* |
| Archived Cycles | No archived cycles exist | No archived cycles available. | View Active Cycles | — |
| Cycle Details → Issues | The cycle has no assigned issues | No issues have been added to this cycle yet. | Create Issue | — |
| Cycle Activity | No cycle activity has been recorded | No activity yet. Updates will appear here as work progresses. | — | — |

---
## 6.6 Issues

| Screen / Section | Condition | Message | Primary Action | Secondary Action |
|------------------|-----------|---------|----------------|------------------|
| Issues List | No issues exist in the workspace | No issues yet. Create your first issue to start tracking work. | Create Issue | — |
| Issues List | No issues match the current search or filters | No matching issues found. Try adjusting your search or filters. | Clear Filters | Create Issue |
| Issue Kanban Column | No matching Issues have the column's status | No issues in this status. Users with update permission can drag an issue here to update its status. | — | — |
| Archived Issues | No archived issues exist | No archived issues available. | View Active Issues | — |
| Backlog | No issues are in the Backlog | Your backlog is empty. Create a new issue or reprioritize existing work. | Create Issue | View All Issues |
| Blocked Issues | No active issues are marked as blocked | No blocked issues. Work can continue across all active issues. | View All Issues | — |
| Todo | No issues are in Todo | No work is ready to start. Move issues into Todo to begin planning. | View Backlog | — |
| In Progress | No issues are currently in progress | Nothing is being worked on right now. Move an issue into progress to begin. | View Todo | — |
| Done | No completed issues yet | Completed work will appear here once issues are finished. | View In Progress | — |
| Issue Activity | No activity has been recorded | No activity yet. Updates will appear here as changes are made. | — | — |
| Issue History | No history is available | No changes have been recorded for this issue yet. | — | — |
---
## 6.7 Collaboration

| Screen / Section | Condition | Message | Primary Action | Secondary Action |
|------------------|-----------|---------|----------------|------------------|
| Comments Section | The issue has no comments | No discussion yet. Start the conversation by adding the first comment. | Add Comment | — |
| Notifications Panel | The user has no notifications | You're all caught up. New notifications will appear when you're assigned an Issue or mentioned in a comment. | — | — |
| Notification Filter | No notifications match the current filter | No matching notifications found. Try adjusting your filters. | Clear Filters | — |

---
## 6.8 Dashboard

| Screen / Section | Condition | Message | Primary Action | Secondary Action |
|------------------|-----------|---------|----------------|------------------|
| Dashboard *(Owner/Admin)* | The workspace contains no projects, cycles, or issues | Welcome to Shipyard! Start by creating your first project, cycle, or issue. | Create Project | Create Issue |
| Dashboard *(Member)* | The workspace contains no projects, cycles, or issues | Welcome to Shipyard! Create the first issue to start tracking work. | Create Issue | — |
| Assigned Issues | The user has no assigned issues | You don't have any assigned issues. | View All Issues | — |
| Recently Viewed Issues | The user has not viewed any issues yet | Recently viewed issues will appear here as you work. | View All Issues | — |
| Overdue Issues | The user has no overdue issues | Great job! You don't have any overdue issues. | — | — |
| Blocked Issues | The user has no blocked assigned issues | You don't have any blocked work. | View Assigned Issues | — |
| Active Projects *(Owner/Admin)* | No active projects exist | No active projects available. Create a project to start organizing work. | Create Project | — |
| Active Projects *(Member)* | No active projects exist | No active projects are currently available. | — | — |
| Current Active Cycle *(Owner/Admin)* | No active cycle exists | No active cycle is currently running. Create a cycle to begin planning work. | Create Cycle | — |
| Current Active Cycle *(Member)* | No active cycle exists | No active cycle is currently running. | — | — |
| Recently Completed Issues | No issues have been completed | Completed issues will appear here once work is finished. | View All Issues | — |
| Recent Activity Feed | No recent workspace activity exists | No recent activity yet. Updates will appear here as your team collaborates. | — | — |

---
## 6.9 Search

| Screen / Section | Condition | Message | Primary Action | Secondary Action |
|------------------|-----------|---------|----------------|------------------|
| Global Search | No search query entered | Start typing to search for issues, projects, cycles, or members. | — | — |
| Search Results | No resources match the search query | No matching results found. Try using different keywords or filters. | Clear Search | — |
| Filter Panel | No resources match the selected filters | No matching results found. Adjust or clear your filters to continue. | Clear Filters | — |
| Saved Views | No saved views exist | You haven't created any saved views yet. Save your current filters for quick access later. | Save Current View | — |

---
## 6.10 Settings

| Screen / Section | Condition | Message | Primary Action | Secondary Action |
|------------------|-----------|---------|----------------|------------------|
| Profile Picture | The user has not uploaded a profile picture | Add a profile picture to help your teammates recognize you. | Upload Photo | — |
| Workspace Logo | The workspace has no logo | Add a workspace logo to personalize your workspace. | Upload Logo | — |

---
