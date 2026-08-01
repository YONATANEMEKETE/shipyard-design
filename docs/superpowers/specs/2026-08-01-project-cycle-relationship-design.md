# Project–Cycle Relationship Design

## Decision

Projects and Cycles are independent entities within a Workspace. Neither entity owns, contains, or directly references the other.

Issues are the only direct bridge between them:

- Every Project belongs to exactly one Workspace.
- Every Cycle belongs to exactly one Workspace.
- An Issue belongs to exactly one Workspace.
- An Issue may belong to one Project.
- An Issue may belong to one Cycle.
- A Project and Cycle associated with the same Issue must belong to that Issue's Workspace.

## Derived Visibility

A Project may show the Cycles represented by its Issues. This is derived information, calculated from Issues that reference both the Project and a Cycle. It does not create a Project–Cycle relationship.

The interface must describe such information as “Cycles represented in this project” or equivalent. It must not use language such as “project cycles,” “cycles assigned to the project,” or “create a cycle for this project.”

## UX Behavior

- Projects and Cycles remain separate primary navigation sections.
- Project Details focuses on project information, project progress, associated Issues, and project activity.
- Cycle Details focuses on cycle information, cycle progress, associated Issues, statistics, and cycle activity.
- Creating a Cycle is available from the Cycles section or the global Create menu, subject to permissions.
- Creating an Issue from Project Details preselects the Project.
- Creating an Issue from Cycle Details preselects the Cycle.
- When both values are selected, the chosen Project and Cycle must belong to the active Workspace.
- An empty Project does not show a “no cycles” state because Cycles are not children of Projects.

## Documentation Changes

Reconcile the PRD and UX documents by:

1. Making the independent relationship explicit in the PRD and Information Architecture.
2. Removing direct Project–Cycle wording from the Screen Inventory, Empty States, and User Flows.
3. Removing Project Details as an entry point for Create Cycle.
4. Keeping issue-based navigation between Projects and Cycles where the relationship is derived through shared Issues.
5. Updating postconditions so they describe consistency across related Issues, not a direct Project–Cycle association.

## Acceptance Checks

- No requirement states or implies that a Project contains or owns a Cycle.
- No requirement states or implies that a Cycle is assigned directly to a Project.
- Every Project–Cycle connection is explicitly described as derived through Issues.
- Create Cycle has no Project-specific entry point.
- Project empty states do not include a Cycles subsection.
- The relationship rules are consistent across Product and UX documentation.
