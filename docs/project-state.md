
# TeamFlow - Project State

## Current Status
Phase 1 completed: Requirements and initial system documentation.
Phase 2 in progress: Database design.

## Repository
https://github.com/Syndhavi/TeamFlow

## Technology Stack
- React + TypeScript
- Node.js + Express + TypeScript
- PostgreSQL
- Prisma
- Tailwind CSS
- Zod
- JWT authentication

## Completed Work
- Created GitHub repository.
- Cloned repository locally.
- Created README.md.
- Created docs/requirements.md.
- Created docs/architecture.md.
- Created docs/database-design.md.
- Committed and pushed initial documentation.

## Established Design Decisions
- Modular monolith architecture.
- User can belong to multiple workspaces.
- Workspace-specific roles: Admin, Manager, Member.
- Membership is a separate entity.
- Workspace contains multiple projects.
- Project belongs to one workspace.
- Task has one assignee initially.
- Projects are archived rather than normally deleted.
- Authorization checks workspace membership and permissions.

## Current Working Stage
Database Design - Stage 1 and Stage 2.

Identified entities:
1. User
2. Workspace
3. Membership
4. Invitation
5. Project
6. Task
7. Comment
8. ActivityLog
9. Notification

## Next Task
Define attributes for:
1. User
2. Workspace
3. Membership

Discuss:
- Primary keys
- Foreign keys
- UUID versus auto-increment IDs
- PostgreSQL data types
- NOT NULL and UNIQUE constraints

Then continue with the remaining entities, relationships, ER diagram, normalization, indexes and Prisma implementation.

## Pending Work
- Complete database design.
- Implement PostgreSQL schema using Prisma.
- Develop backend.
- Develop frontend.
- Testing and deployment.

## Mentoring Instructions
Teach concepts before implementation.
Explain decisions and commands.
Let me write the code.
Proceed step by step.
Do not repeat completed work.
