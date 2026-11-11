# TeamFlow — Project State

**Last updated:** 11 November 2026  
**Project status:** Requirements and conceptual database design completed; implementation has not started.

## 1. Project overview

TeamFlow is a collaborative project and task management platform for organizations. It provides workspaces, projects, team membership, task assignment, progress tracking and collaboration.

- **GitHub:** https://github.com/Syndhavi/TeamFlow
- **Local path:** `D:\TeamFlow\TeamFlow`
- **Architecture:** Modular monolith
- **Development approach:** Independent, incremental development with regular meaningful Git commits and pushes.

## 2. Technology stack

### Planned MVP stack
- Frontend: React + TypeScript
- Backend: Node.js + Express.js + TypeScript
- Database: PostgreSQL
- ORM: Prisma
- Styling: Tailwind CSS
- Validation: Zod
- Authentication: JWT with secure refresh-token cookies
- API testing: Postman
- Version control: Git and GitHub

### Planned later additions
Redis, Socket.IO, Docker, automated testing, CI/CD, logging and monitoring. These are future additions, not yet implemented.

## 3. Product scope

The planned application includes:
- User registration, login, logout and profile management.
- Workspace creation, invitations and membership management.
- Workspace-specific roles: Admin, Manager and Member.
- Project creation, updating, manager assignment, archiving and restoration.
- Task creation, optional single assignee, priorities, deadlines, status and filtering.
- Task comments, activity history and notifications.
- Dashboard statistics and overdue-task views.
- Authentication, authorization and workspace-level resource isolation.

## 4. Completed work

### Repository and documentation setup
- Created the GitHub repository.
- Cloned it locally to `D:\TeamFlow\TeamFlow`.
- Created `README.md`.
- Created `docs/requirements.md`.
- Created `docs/architecture.md`.
- Created `docs/database-design.md`.
- Created `docs/project-state.md`.
- Initial documentation commit was pushed to `main`:
  - Commit: `4523942`
  - Message: `Add Initial Project Documentation`

### Requirements and system design
- Defined the product overview, problem statement, objectives and target users.
- Identified functional requirements for authentication, workspaces, projects, tasks, collaboration and dashboard.
- Identified non-functional requirements for security, performance, reliability, maintainability and scalability.
- Selected a modular monolith rather than microservices.

### Conceptual database design
The nine planned entities have been identified and documented:
1. User
2. Workspace
3. Membership
4. Invitation
5. Project
6. Task
7. Comment
8. ActivityLog
9. Notification

The current `docs/database-design.md` contains the proposed attributes and PostgreSQL types, relationships/cardinality, UUID strategy, normalization considerations, constraints, deletion/archival policies, authorization rules, indexes, important transactions and future extensions.

**Database design status:** Conceptual design finalized. It has not yet been translated into executable PostgreSQL DDL or Prisma models, and implementation-level details still need review during schema implementation.

## 5. Finalized product and data-design decisions

### Workspace and membership
- A user may belong to multiple workspaces.
- A user's role is specific to each workspace.
- `Membership` is a separate entity connecting `User` and `Workspace`.
- Roles: `ADMIN`, `MANAGER`, `MEMBER`.
- A workspace may initially contain only its creator.
- Workspace creation and creation of the creator's Admin membership must happen atomically in one transaction.
- When a user leaves a workspace, preserve the membership record and mark it `LEFT`; do not hard-delete it.
- The initial design uses one membership record per user/workspace pair. Rejoining reactivates that record. A separate membership-history table may be added later if full repeated join/leave history is required.

### Projects and tasks
- A workspace can contain multiple projects; each project belongs to exactly one workspace.
- Project manager assignment is optional, but an assigned manager must be an active member of the project's workspace.
- Projects are archived rather than normally hard-deleted.
- Archiving preserves tasks, comments and activity history.
- A task belongs to exactly one project.
- A task may be unassigned or have one assignee.
- An assignee must be an active member of the workspace containing the task's project when assigned.
- If an assigned user leaves the workspace, retain their existing task assignments. An authorized manager must explicitly reassign outstanding work when appropriate.
- Multiple task assignees are out of initial scope.

### Comments and collaboration
- A task can have multiple comments; each comment has one author.
- Authors may edit and delete their own comments.
- Comment deletion uses soft deletion (`deleted_at`) so historical records are retained.
- Comment edits and deletions should create activity records.
- Authorized workspace administrators may be permitted to moderate comments; exact permission policy will be specified during authorization design.

### Security and integrity
- Use UUID primary keys. UUIDs reduce guessability but are not an authorization mechanism.
- Store encoded Argon2id or bcrypt password hashes, never plaintext passwords.
- Normalize email addresses before enforcing uniqueness.
- Protected operations must authenticate the requester and validate resource/workspace membership and permissions.
- Project manager and task assignee membership rules must be enforced by backend business logic; database constraints may strengthen integrity where practical.
- `ActivityLog` uses `entity_type` and `entity_id` as a polymorphic reference; this is not a conventional foreign key.
- `Notification` uses explicit nullable references to supported resource types (task, project or comment), with a rule that at most one resource reference is populated.
- Store timestamps using PostgreSQL `TIMESTAMPTZ`.
- Use transactions for multi-record operations such as workspace creation, invitation acceptance, task assignment with activity/notification creation, and project archiving.

## 6. Current repository snapshot

The uploaded project archive contains:
- `README.md`
- `docs/requirements.md`
- `docs/architecture.md`
- `docs/database-design.md`
- `docs/project-state.md`

The archive does not currently contain `client/`, `server/`, Prisma schema/migrations, application source code, tests or deployment configuration.

**Important observation:** `docs/architecture.md` is currently empty. The architecture decisions exist in the project handoff/context, but the architecture document still needs to be written before or during implementation planning.

The README currently describes the project as being in the requirements and system-design phase; update that status to reflect the completed conceptual database design when next editing the README.

## 7. Current phase and exact position

### Phase 1 — Requirements and initial system design
**Status:** Completed at the conceptual/documentation level.

### Phase 2 — Database design
**Status:** Conceptual design completed; implementation review remains.

Completed:
- Entity identification.
- Initial attributes and PostgreSQL data types.
- Relationship and cardinality design.
- Conceptual ER diagram.
- Normalization review.
- Primary-key, unique, foreign-key and CHECK constraint planning.
- Deletion, archival and membership lifecycle policies.
- Authorization and cross-workspace integrity rules.
- Initial index strategy.
- Multi-record transaction workflows.
- Final product decisions for workspace creation, unassigned tasks, departed-member assignments and comment editing/deletion.

Not yet completed:
- Review and reconcile every conceptual decision for Prisma/PostgreSQL implementation.
- Decide exact Prisma relation names, referential actions and enum definitions.
- Decide whether composite foreign keys will be implemented for workspace membership integrity or enforced only in service logic.
- Resolve implementation details for case-insensitive email uniqueness and partial unique invitation index.
- Verify consistency of the ER diagram, attribute tables and constraints.
- Write the final architecture document.
- Implement PostgreSQL/Prisma schema and migrations.

## 8. Recommended next steps

Proceed in this order. Do not jump directly into application feature coding.

1. **Review the existing `docs/database-design.md`**
   - Check entity attributes, nullability, relationships and foreign keys.
   - Resolve any contradictions or missing referential actions.
   - Confirm the notification resource-reference CHECK constraint.
   - Confirm membership reactivation semantics and invitation uniqueness.
2. **Complete `docs/architecture.md`**
   - Document frontend, API, backend modules, service/repository responsibilities, PostgreSQL/Prisma and request flow.
   - Include authentication and workspace authorization boundaries.
3. **Prepare the implementation-level schema**
   - Define Prisma enums and models.
   - Define relation fields and inverse relations.
   - Define defaults, unique constraints and indexes.
   - Identify database-specific partial indexes or CHECK constraints that need custom SQL migrations.
4. **Set up the development environment**
   - Confirm Node.js/npm and PostgreSQL installation.
   - Create the frontend and backend project structure.
   - Configure environment variables safely; never commit `.env` or secrets.
5. **Implement the initial Prisma schema**
   - Run formatting and Prisma validation.
   - Review the generated migration before applying it.
   - Verify tables and constraints in PostgreSQL.
6. **Begin backend development**
   - Start with application configuration, database connection, health endpoint and error handling.
   - Then implement authentication and workspace modules in small steps.

The next immediate task is **database-design implementation review**, not writing application features. First inspect the current `docs/database-design.md` and identify any unresolved design or Prisma mapping questions.

## 9. Git and project workflow

Local working directory: `D:\TeamFlow\TeamFlow`  
Primary branch: `main`

Use this workflow for meaningful completed units of work:

1. Check repository state with `git status`.
2. Review the files changed.
3. Stage only the intended files, for example `git add docs/database-design.md docs/project-state.md`.
4. Review staged changes with `git diff --cached`.
5. Commit with a meaningful message, for example `git commit -m "Finalize database design documentation"`.
6. Push with `git push origin main`.
7. Confirm the commit appears on GitHub.

Do not commit `.env`, passwords, API keys, database credentials or other secrets. Do not make a commit claim until the user confirms the command succeeded.

## 10. Mandatory project-state update procedure

**Update `docs/project-state.md` whenever a meaningful design decision is finalized or a valid implementation milestone is completed.**

For every finalized design decision:
- Add it to the relevant section of this file.
- Update the relevant design document (requirements, architecture or database design).
- Record the date and rationale when the decision materially affects architecture, data integrity, security or scope.
- If a decision replaces an earlier one, explicitly mark the old decision as superseded rather than leaving contradictory instructions.

For every valid implementation milestone:
- Record what was implemented and verified.
- List files created or modified.
- Record relevant commands/tests and their outcomes.
- Record the Git commit hash only after a successful commit.
- State what remains incomplete and the exact next task.

At the end of each meaningful session, the mentor must provide a **Project Checkpoint** with:
1. What was completed.
2. Concepts learned.
3. Decisions finalized.
4. Files created or modified.
5. Git commit made (or explicitly state that no commit was made).
6. Remaining work.
7. Exact next task.

The user will then update `docs/project-state.md`, commit and push it. Provide ready-to-paste Markdown for the state update whenever a milestone or design decision is reached. Do not assume the state file has been updated or pushed until the user confirms.

## 11. Mentoring instructions

- Act as a technical mentor, not as someone who silently completes the application.
- The user knows DBMS fundamentals; focus on applying theory to engineering decisions rather than repeating elementary definitions.
- Explain the reasoning and trade-offs behind design and implementation choices.
- Break work into small, manageable steps.
- Explain every command, including Git commands.
- Let the user write code; review it and help debug.
- Explain errors and their causes.
- Do not skip intermediate steps or assume unfamiliar tools are already understood.
- Do not rush or jump ahead before the current stage is understood.
- The user generally has around 2–3 hours per day, with a flexible schedule.
- At the beginning of a new session, briefly summarize the current stage and the immediate next task.
