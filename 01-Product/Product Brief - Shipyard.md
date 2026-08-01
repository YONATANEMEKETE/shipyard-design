# Product Brief

## Project

- **Working Name:** Shipyard | Plan. Build. Ship.
- **Category:** Open-Source B2B SaaS
- **Type:** Software Project Management Platform

## 1. Product

We are building a modern, open-source project management and issue tracking platform designed specifically for small software engineering teams (2-30 people).

Inspired by Linear, the platform helps teams plan, organize, and ship software through a fast, intuitive, and developer-first experience. Rather than competing on feature count, the product focuses on delivering the essential workflows required by small engineering teams while avoiding unnecessary enterprise complexity.

The goal is to create a production-grade application that demonstrates modern full-stack engineering practices while providing a practical tool that teams can adopt in real-world projects.

## 2. Problem

Software teams need a simple and efficient way to plan, organize, and track development work.

Many existing project management tools fall into one of two categories:

- Too simple to support growing engineering teams.
- Too complex, requiring significant configuration and administration.

For small software teams, this creates unnecessary overhead. Time is spent managing the project management tool instead of building software.

Our product addresses this by providing the essential software development workflows without enterprise-level complexity.

## 3. Target Users

### Primary Users

**Startup Engineering Teams (2-30 people)**

Small engineering teams building SaaS products that need a structured yet lightweight project management platform.

### Secondary Users

- Indie Hackers
- Solo Developers
- Small Software Agencies

### Non-Target Users

- Large enterprises
- Non-technical business teams
- HR, Sales, Finance, and CRM workflows
- Organizations requiring extensive governance and compliance

## 4. Existing Solutions

### Primary Competitors

- Linear
- Jira
- Trello
- GitHub Projects

### Positioning

Rather than replacing enterprise tools, we aim to become the preferred project management platform for small software teams that value simplicity, speed, and developer experience.

## 5. Why This Product

We believe small software teams deserve a project management platform that is:

- Fast
- Opinionated
- Easy to learn
- Focused on software development

Instead of maximizing customization, we prioritize excellent defaults and streamlined workflows.

Internally, this project also serves as a production-grade learning platform and an open-source foundation for future Build Stage projects.

## 6. Core Value

### Core Value Statement

Help small software teams plan, organize, and ship software faster through a simple, developer-first project management experience.

### Value Pillars

- Simplicity
- Speed
- Developer-First Experience
- Visibility
- Open Source

### Product Promise

The easiest way for small software teams to manage and ship software.

## 7. Core Workflows

### 1. Manage Work

- Create issues
- Assign work
- Prioritize tasks
- Track status
- Complete issues

### 2. Plan Projects

- Create projects
- Organize related work
- Monitor project progress
- Delete projects without deleting their issues

### 3. Run Development Cycles

- Create cycles
- Schedule cycles without overlapping dates
- Plan work
- Track sprint progress
- Complete cycles

### 4. Collaborate

- Comment on issues
- Mention teammates
- Review activity history
- Receive notifications

### 5. Track Progress

- Dashboard overview
- Project progress
- Cycle status
- Assigned work
- Blocked issues

## 8. MVP Scope

### Must Have

#### Authentication & Workspace

- Authentication
- Workspace creation
- Workspace archiving and restoration
- Workspace ownership transfer
- Team invitations
- User profiles
- Theme preference

#### Issue Management

- CRUD issues
- Issue archiving and restoration
- Statuses
- Priorities
- Assignees
- Labels
- Due dates
- Blocked flag with an optional reason
- Search & filtering

#### Projects

- Project management
- Project archiving and restoration
- Permanent project deletion with automatic issue unassignment
- Issue grouping
- Progress tracking

#### Cycles

- Create cycles
- Start, complete, and reopen cycles through controlled lifecycle actions
- Prevent date overlap between non-archived cycles
- Assign issues
- Track cycle progress
- Complete cycles
- Archive and restore cycles
- Delete future planned cycles

#### Collaboration

- Comments
- Mentions
- Activity history

#### Dashboard

- My Issues
- Active Projects
- Current Cycle
- Recent Activity

#### Notifications

- In-app notifications
- Assignment notifications
- Mention notifications

### Future Enhancements

- Kanban Board
- Calendar View
- Timeline View
- Keyboard Shortcuts
- Command Palette
- GitHub Integration
- Slack Integration
- Analytics
- Templates
- Recurring Issues

## 9. Out of Scope

The initial product will intentionally exclude:

- Enterprise administration
- Advanced permission systems
- SSO
- Audit logs
- CRM
- HR features
- Finance features
- Billing
- AI-powered workflows
- Portfolio management
- Resource planning
- Heavy customization

These may be considered in future versions but are outside the scope of the MVP.

## 10. Success Metrics

### Product Success

The MVP is successful if a small software team can use it as their primary engineering project management platform.

Success indicators include:

- Workspace setup in under five minutes.
- Team onboarding within thirty minutes.
- Ability to manage an entire development cycle.
- Smooth execution of all core workflows.
- Users describe the product as simple, fast, and intuitive.

### Engineering Success

The platform should demonstrate production-grade quality through:

- Modular architecture
- Strong TypeScript coverage
- Secure authentication
- Accessible UI
- High performance
- Reliable APIs
- Automated testing
- CI/CD pipeline
- Production deployment
- Clear documentation

### Community Success

As an open-source project, success also includes:

- Excellent documentation
- Easy local setup
- Contributor guidelines
- Transparent issue management
- Community contributions

## Product Principles

The following principles guide every product decision.

### 1. Simplicity over Complexity

Every feature should reduce friction for small engineering teams.

### 2. Build for Software Teams First

Optimize for software development workflows rather than general business management.

### 3. Compete Through Focus, Not Features

Deliver exceptional core workflows instead of feature parity.

### 4. Opinionated by Default

Provide sensible defaults rather than extensive configuration.

### 5. Every Feature Must Strengthen the Core Value

Features should help teams plan, organize, or ship software faster.

### 6. Every Feature Needs a Reason

Every addition must solve a real user problem and align with the product philosophy.

### 7. Quality Over Quantity

A small number of polished features is better than a large number of mediocre ones.

## Design Principles

The user experience should follow these principles throughout the product.

- Speed First
- Opinionated Defaults
- Developer-Centric
- Consistency
- Clarity Over Decoration
- Keyboard Friendly

## Vision Statement

To become the leading open-source project management platform for small software teams by delivering a fast, opinionated, and developer-first experience that helps teams ship software with confidence.
