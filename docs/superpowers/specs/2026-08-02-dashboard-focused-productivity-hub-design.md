# Dashboard Focused Productivity Hub Design

Date: 2026-08-02  
Status: Approved visual direction; pending user review

## Purpose

The default workspace Dashboard helps a logged-in user answer one question immediately: **What should I work on next?** It prioritizes the Software Engineer workflow while preserving enough cycle and project context for Engineering Leads and Founders.

The approved visual reference is [`03-UI/references/dashboard-focused-productivity-hub.png`](../../../03-UI/references/dashboard-focused-productivity-hub.png).

## Scope

Update the main content of the existing `Dashboard / Default` frame in `03-UI/shipyard.pen`. Preserve the application shell, sidebar, global header, greeting, urgent-work banner, and four-number issue summary strip.

Add three content areas below the summary strip:

1. A dominant tabbed **My work** issue list.
2. A secondary right column containing **Current cycle** and **Active projects**.
3. A tertiary full-width **Recent activity** feed.

Do not introduce analytics, configurable widgets, separate role-specific dashboards, or a permanent quick-actions card. This canvas update covers the populated default state only; separate empty, loading, error, and permission-state frames are outside its scope.

## Information Hierarchy

### 1. Existing top area

Keep these elements unchanged:

- Personalized greeting and date summary.
- Urgent-work banner with one highest-priority issue and a review action.
- Four-number strip for Assigned issues, Overdue, Blocked, and Completed this week.

### 2. My work

`My work` is the primary dashboard panel and occupies roughly 65% of the first content row.

- Header: title, supporting description, and `View all` link.
- Tabs: `Assigned` (default), `Created`, and `Recently viewed`.
- Five compact issue rows.
- Each row shows priority, title, issue key and project, urgency or due-date context, and workflow status.
- Selecting an issue opens Issue Details.
- `View all` opens the relevant Issues view while preserving the active tab's meaning.

The visual hierarchy must make this panel more prominent than cycle, project, and activity information.

### 3. Current cycle

The Current Cycle card is compact and status-oriented rather than analytical.

- Title: `Current cycle`.
- Cycle name and date range.
- Segmented donut chart with small gaps and visibly rounded arc caps.
- Donut center shows the completion percentage and `complete` label.
- Segments represent Completed, In progress, and Remaining using restrained semantic colors.
- Supporting details show completed issue count, total issue count, and days remaining.
- A compact legend identifies all three segments without relying on color alone.
- Selecting the card opens Cycle Details.

Do not add axes, trends, velocity, burndown, forecasts, gradients, or decorative chart effects.

### 4. Active projects

The Active Projects card sits below Current Cycle in the secondary column.

- Header with `View all` link.
- Up to three project rows.
- Each row shows project identity, health label, completion percentage with a small progress bar, and Project Owner avatar.
- Selecting a row opens Project Details.
- `View all` opens the Active Projects section.

### 5. Recent activity

Recent Activity spans the full content width beneath the primary row.

- Header with `View all` link.
- Four recent workspace events arranged as the approved compact two-column feed.
- Each item shows a semantic icon or member avatar, a concise action sentence, related resource context, and relative time.
- Activity is reverse chronological.
- Selecting an item opens the related resource.

## Layout and Visual Rules

- Preserve the existing 1440px desktop frame and Shipyard app shell.
- Use the current warm off-white background, white surfaces, thin neutral borders, restrained Harbor Amber accent, compact typography, and existing spacing tokens.
- First content row: 65% My Work and 35% secondary column, adjusted only as needed to align with the existing spacing grid.
- Secondary column: Current Cycle above Active Projects.
- Recent Activity: full width below the first content row.
- Prefer dense lists and direct alignment over nested cards.
- Use minimal shadow, no gradients, and restrained corner radii consistent with the existing design system.
- Reuse existing components and semantic variables before creating new primitives.

## Permissions and Actions

- Dashboard content is scoped to the active workspace.
- Every role can view the Dashboard and create Issues through the existing global Create menu.
- Owners and Admins additionally receive Project and Cycle creation actions from the existing global menu and permission-aware empty states.
- Unauthorized actions are omitted rather than disabled.
- No permanent Quick Actions panel is added because global Search and Create already provide those actions, while My Work provides direct access to assigned issues.

## Data and Interaction Model

- The Dashboard loads personal issue data, active cycle data, active project summaries, and recent activity independently.
- Failure in one content area must not block the other areas.
- Tabs update only the My Work list; they do not reload or reorder the rest of the page.
- Archived resources are excluded.
- Issue, Cycle, Project, and Activity items link to their existing detail surfaces.
- Activity is ordered newest first.

## Empty, Loading, and Error States

- **My work:** explain that the user has no matching issues and link to All Issues.
- **Current cycle:** Owners/Admins may create a cycle; Members see a neutral no-active-cycle message.
- **Active projects:** Owners/Admins may create a project; Members see a neutral no-active-projects message.
- **Recent activity:** explain that updates will appear as the team works.
- **Loading:** each panel uses a stable skeleton matching its final dimensions.
- **Partial failure:** retain available panels and show a local retry action only in the failed panel.
- **Full failure:** preserve the app shell and provide a Dashboard-level retry.

## Accessibility

- All tabs, issue rows, project rows, cycle navigation, and `View all` links are keyboard accessible.
- The active tab is conveyed by text/state, not color alone.
- Donut segments have a text legend and numeric completion value.
- Project health and issue status are expressed with text labels, not color alone.
- Interactive rows retain visible focus states and adequate target sizes.
- Text and semantic colors meet the contrast rules in `03-UI/design.md`.

## Acceptance Criteria

1. The selected `Dashboard / Default` frame retains the approved top area and application shell.
2. My Work is the most prominent panel and includes the three approved tabs and five issue rows.
3. Current Cycle uses a segmented donut with rounded arc ends, a centered percentage, and a text legend.
4. Active Projects displays no more than three project summaries.
5. Recent Activity displays four newest-first events.
6. The page contains no analytics chart, burndown graph, separate quick-actions card, clipping, collapsed layout, or overlapping content.
7. All permission-sensitive and empty-state actions follow the existing Shipyard role rules.
8. Pencil structural checks report no layout problems, and the final frame matches the approved visual hierarchy.

## Verification

- Inspect the final Pencil hierarchy and bounds for clipping, overlap, and unsupported properties.
- Capture a final screenshot of the complete Dashboard frame.
- Compare the screenshot with the approved visual reference for hierarchy, content, spacing, and donut treatment.
- Verify the populated default state in this canvas update. Use the state behavior defined above when separate state frames or engineering implementation are planned.
