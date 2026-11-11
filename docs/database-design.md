# TeamFlow — Database Design

**Project:** TeamFlow  
**Database:** PostgreSQL  
**ORM:** Prisma  
**Architecture:** Modular Monolith  
**Design status:** Final conceptual design — pending implementation review

---

## 1. Database overview

TeamFlow is a collaborative project and task management platform. Its database supports user accounts, workspaces, memberships, projects, tasks and team collaboration.

The initial database contains nine main entities:

1. User
2. Workspace
3. Membership
4. Invitation
5. Project
6. Task
7. Comment
8. ActivityLog
9. Notification

### Design principles

- Use UUIDs as primary keys.
- Use foreign keys to maintain referential integrity.
- Use workspace-specific memberships and roles.
- Preserve historical project, task and collaboration data.
- Use transactions for operations involving multiple related records.
- Enforce authorization in the backend service layer.
- Use indexes based on expected query patterns.
- Avoid unnecessary denormalization.

## 2. Entity attributes

### 2.1 User

Stores information about registered users.

| Attribute | Data type | Constraints |
|---|---|---|
| id | UUID | Primary key |
| name | VARCHAR(100) | NOT NULL |
| email | VARCHAR(255) | NOT NULL, UNIQUE |
| password_hash | TEXT | NOT NULL |
| avatar_url | TEXT | NULL |
| is_active | BOOLEAN | NOT NULL, DEFAULT TRUE |
| email_verified_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Design decisions:
- Email addresses are normalized before storage, preferably to lowercase.
- Email must be unique.
- Passwords are stored as encoded Argon2id or bcrypt hashes, never plaintext.
- Avatar URL and email verification timestamp are optional.
- Deactivation is preferred over immediate permanent account deletion.

### 2.2 Workspace

Represents an organization or team environment.

| Attribute | Data type | Constraints |
|---|---|---|
| id | UUID | Primary key |
| name | VARCHAR(150) | NOT NULL |
| slug | VARCHAR(180) | NOT NULL, UNIQUE |
| description | TEXT | NULL |
| created_by | UUID | NOT NULL, FK → User.id |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Design decisions:
- A workspace can initially contain only its creator.
- Its creator automatically receives an Admin membership.
- Workspace creation and initial membership creation must happen in one transaction.
- Slugs are unique and can be used in workspace URLs.

### 2.3 Membership

Connects users to workspaces and stores workspace-specific roles.

| Attribute | Data type | Constraints |
|---|---|---|
| id | UUID | Primary key |
| user_id | UUID | NOT NULL, FK → User.id |
| workspace_id | UUID | NOT NULL, FK → Workspace.id |
| role | MembershipRole | NOT NULL |
| status | MembershipStatus | NOT NULL |
| joined_at | TIMESTAMPTZ | NOT NULL |
| left_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Enums:

- MembershipRole: ADMIN, MANAGER, MEMBER
- MembershipStatus: ACTIVE, LEFT

Constraints:
- UNIQUE(user_id, workspace_id)
- An ACTIVE membership must have `left_at IS NULL`.
- A LEFT membership must have `left_at IS NOT NULL`.

Design decisions:
- A user can belong to multiple workspaces.
- Each workspace can contain multiple users.
- The same user can have different roles in different workspaces.
- Membership records are not deleted when users leave.
- When a user rejoins, their existing membership record is reactivated and its membership dates are updated.
- If a complete history of repeated joining and leaving is required later, a separate MembershipHistory table can be introduced.

### 2.4 Invitation

Stores invitations sent to people to join workspaces.

| Attribute | Data type | Constraints |
|---|---|---|
| id | UUID | Primary key |
| workspace_id | UUID | NOT NULL, FK → Workspace.id |
| email | VARCHAR(255) | NOT NULL |
| invited_by | UUID | NOT NULL, FK → User.id |
| role | MembershipRole | NOT NULL |
| token_hash | TEXT | NOT NULL, UNIQUE |
| status | InvitationStatus | NOT NULL |
| expires_at | TIMESTAMPTZ | NOT NULL |
| accepted_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Enum:
- InvitationStatus: PENDING, ACCEPTED, EXPIRED, REVOKED

Design decisions:
- Invitations can be sent to users who have not registered.
- Email is stored instead of requiring an existing user ID.
- The invitation token is stored as a hash.
- Multiple historical invitations can exist for the same email and workspace.
- Only one pending invitation should exist for a given workspace and normalized email.
- Expired or revoked invitations cannot be accepted.
- The accepting user's verified email must match the invitation email.

### 2.5 Project

Represents an initiative within a workspace.

| Attribute | Data type | Constraints |
|---|---|---|
| id | UUID | Primary key |
| workspace_id | UUID | NOT NULL, FK → Workspace.id |
| name | VARCHAR(150) | NOT NULL |
| description | TEXT | NULL |
| manager_id | UUID | NULL, FK → User.id |
| status | ProjectStatus | NOT NULL |
| start_date | DATE | NULL |
| due_date | DATE | NULL |
| is_archived | BOOLEAN | NOT NULL, DEFAULT FALSE |
| archived_at | TIMESTAMPTZ | NULL |
| created_by | UUID | NOT NULL, FK → User.id |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Enum:
- ProjectStatus: PLANNED, ACTIVE, COMPLETED, ON_HOLD

Constraints:
- due_date must not be earlier than start_date when both are present.
- `is_archived = FALSE` requires `archived_at IS NULL`.
- `is_archived = TRUE` requires `archived_at IS NOT NULL`.

Design decisions:
- Every project belongs to exactly one workspace.
- A workspace can contain multiple projects.
- A project manager must be an active member of the project's workspace when assigned.
- Manager assignment is optional.
- Projects are archived rather than permanently deleted during normal application use.
- Archived projects retain their tasks, comments and activity history.

### 2.6 Task

Represents an individual unit of work within a project.

| Attribute | Data type | Constraints |
|---|---|---|
| id | UUID | Primary key |
| project_id | UUID | NOT NULL, FK → Project.id |
| title | VARCHAR(200) | NOT NULL |
| description | TEXT | NULL |
| assignee_id | UUID | NULL, FK → User.id |
| created_by | UUID | NOT NULL, FK → User.id |
| status | TaskStatus | NOT NULL |
| priority | TaskPriority | NOT NULL |
| due_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |

Enums:

TaskStatus:
- TODO
- IN_PROGRESS
- IN_REVIEW
- DONE

TaskPriority:
- LOW
- MEDIUM
- HIGH
- URGENT

Design decisions:
- A task belongs to exactly one project.
- A project can contain multiple tasks.
- A task can have zero or one assignee.
- Unassigned tasks are allowed.
- The assignee must belong to the workspace containing the task's project when assigned.
- Multiple assignees are not supported initially.
- If an assigned user leaves a workspace, their existing task assignments are retained.
- An authorized manager must reassign outstanding work when appropriate.
- Leaving a workspace does not automatically delete or modify task records.

### 2.7 Comment

Stores comments associated with tasks.

| Attribute | Data type | Constraints |
|---|---|---|
| id | UUID | Primary key |
| task_id | UUID | NOT NULL, FK → Task.id |
| author_id | UUID | NOT NULL, FK → User.id |
| content | TEXT | NOT NULL |
| created_at | TIMESTAMPTZ | NOT NULL |
| updated_at | TIMESTAMPTZ | NOT NULL |
| deleted_at | TIMESTAMPTZ | NULL |

Design decisions:
- A task can have multiple comments.
- Every comment has one author.
- Authors can edit and delete their own comments.
- Authorized workspace administrators may be allowed to moderate comments.
- Deletion uses soft deletion through `deleted_at`.
- Deleted comments are hidden from normal views, while their historical record is retained.
- Comment edits and deletions should generate activity records.

### 2.8 ActivityLog

Records important actions performed inside workspaces.

| Attribute | Data type | Constraints |
|---|---|---|
| id | UUID | Primary key |
| workspace_id | UUID | NOT NULL, FK → Workspace.id |
| actor_user_id | UUID | NULL, FK → User.id |
| action | VARCHAR(100) | NOT NULL |
| entity_type | VARCHAR(50) | NOT NULL |
| entity_id | UUID | NOT NULL |
| metadata | JSONB | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Example actions:
- PROJECT_CREATED
- PROJECT_ARCHIVED
- TASK_CREATED
- TASK_ASSIGNED
- TASK_STATUS_UPDATED
- COMMENT_ADDED
- COMMENT_EDITED
- COMMENT_DELETED

Design decisions:
- Activity records belong to a workspace.
- Actor can be NULL if the original user account is later anonymized or removed.
- `entity_type` and `entity_id` form a polymorphic reference.
- This reference does not have a conventional foreign key to all possible entity tables.
- Metadata stores action-specific information, such as previous and new task assignees.
- Activity records should be preserved for audit history.

### 2.9 Notification

Stores notifications for individual users.

| Attribute | Data type | Constraints |
|---|---|---|
| id | UUID | Primary key |
| recipient_user_id | UUID | NOT NULL, FK → User.id |
| workspace_id | UUID | NULL, FK → Workspace.id |
| type | VARCHAR(50) | NOT NULL |
| title | VARCHAR(200) | NOT NULL |
| message | TEXT | NOT NULL |
| task_id | UUID | NULL, FK → Task.id |
| project_id | UUID | NULL, FK → Project.id |
| comment_id | UUID | NULL, FK → Comment.id |
| read_at | TIMESTAMPTZ | NULL |
| created_at | TIMESTAMPTZ | NOT NULL |

Design decisions:
- Every notification belongs to one recipient.
- A notification may optionally reference a workspace.
- A notification may reference a task, project or comment.
- At most one of task_id, project_id and comment_id can be populated.
- `read_at IS NULL` means the notification is unread.
- A notification can also exist without a resource reference, for general workspace notifications.

---

## 3. Relationships and cardinality

| Parent | Child | Cardinality |
|---|---|---|
| User | Membership | 1:N |
| Workspace | Membership | 1:N |
| User | Invitation | 1:N |
| Workspace | Invitation | 1:N |
| User | Workspace (creator) | 1:N |
| Workspace | Project | 1:N |
| User | Project (manager) | 1:N |
| User | Project (creator) | 1:N |
| Project | Task | 1:N |
| User | Task (assignee) | 1:N |
| User | Task (creator) | 1:N |
| Task | Comment | 1:N |
| User | Comment | 1:N |
| Workspace | ActivityLog | 1:N |
| User | ActivityLog | 1:N |
| User | Notification | 1:N |
| Workspace | Notification | 1:N, optional |

### Important many-to-many relationship

User and Workspace have a many-to-many relationship.

It is implemented through Membership:

- One user can have many memberships.
- One workspace can have many memberships.
- Each membership belongs to exactly one user and one workspace.

The Membership table also stores the user's role and membership status.

---

## 4. Primary keys and identifiers

All main entities use UUID primary keys.

Reasons:
- Large identifier space.
- Difficult-to-guess resource identifiers.
- Convenient generation across distributed or independently running application components.
- Avoid exposing sequential record counts.

UUID collisions are extraordinarily unlikely with proper generation. PostgreSQL primary-key constraints provide a final uniqueness guarantee.

In Prisma, UUIDs can be generated with `@default(uuid())`.

UUIDs are not an authorization mechanism. All resource access must still be validated by the backend.

---

## 5. Normalization

The proposed design follows the principles of 1NF, 2NF and 3NF.

### First Normal Form
- Store atomic values.
- Do not store multiple workspace IDs in a single User column.
- Use Membership to represent user-workspace associations.

### Second Normal Form
- Workspace-specific roles depend on the combination of user and workspace.
- Membership stores this relationship-specific information.

### Third Normal Form
- Workspace attributes belong in Workspace rather than being duplicated in Project.
- Project attributes belong in Project rather than being duplicated in Task.
- Related entities are referenced using foreign keys.

JSONB is used only for variable activity metadata, not to replace the structured relational model.

---

## 6. Constraints and integrity rules

### Unique constraints
- User.email
- Workspace.slug
- Membership(user_id, workspace_id)
- Invitation.token_hash

Additional partial unique index:
- At most one pending invitation for a normalized email and workspace.

### NOT NULL constraints

Required attributes include:
- User.name, User.email, User.password_hash
- Workspace.name, Workspace.slug, Workspace.created_by
- Membership.user_id, Membership.workspace_id, Membership.role, Membership.status
- Invitation.workspace_id, Invitation.email, Invitation.token_hash, Invitation.expires_at
- Project.workspace_id, Project.name, Project.created_by
- Task.project_id, Task.title, Task.created_by, Task.status, Task.priority
- Comment.task_id, Comment.author_id, Comment.content
- ActivityLog.workspace_id, ActivityLog.action, ActivityLog.entity_type, ActivityLog.entity_id
- Notification.recipient_user_id, Notification.type, Notification.title, Notification.message

### CHECK constraints
- Project due date must not be earlier than its start date.
- Active membership must have no left_at timestamp.
- Left membership must have a left_at timestamp.
- Archived project must have archived_at populated.
- Non-archived project must have archived_at NULL.
- Notification must not reference multiple resource types simultaneously.

### Referential integrity
- Foreign keys protect ordinary parent-child relationships.
- Membership and resource authorization rules are additionally enforced by backend services.
- Polymorphic ActivityLog references are not ordinary foreign keys.

---

## 7. Deletion and archival policies

| Relationship | Proposed policy |
|---|---|
| User → Membership | Preserve membership; deactivate or mark LEFT |
| Workspace → Membership | Restrict hard deletion |
| Workspace → Project | Restrict hard deletion |
| Project → Task | Restrict hard deletion |
| Task → Comment | Preserve comments; use soft deletion where appropriate |
| User → Project manager | SET NULL if user is permanently removed |
| User → Task assignee | SET NULL only for permanent removal; not when leaving a workspace |
| User → Task creator | SET NULL for permanent removal |
| User → Comment author | SET NULL for permanent removal or anonymize |
| User → ActivityLog actor | SET NULL or anonymize |
| User → Notification recipient | CASCADE when permanently deleting the account, subject to retention requirements |

Normal application workflows should use deactivation, leaving and archiving rather than hard deletion.

Permanent account deletion must be designed with privacy, retention and audit requirements in mind.

---

## 8. Authorization and cross-workspace integrity

All protected resource operations must validate:

1. The requester is authenticated.
2. The requested resource exists.
3. The resource belongs to the expected workspace.
4. The requester has an ACTIVE membership in that workspace.
5. The requester's role permits the operation.
6. Any additional resource-level permission is satisfied.

Additional rules:
- Project manager must be an active workspace member when assigned.
- Task assignee must be an active member of the project's workspace when assigned.
- A user who leaves a workspace cannot continue accessing its resources through their old membership.
- Existing task assignments remain intact after a member leaves.
- Reassignment requires an authorized manager or another user with the appropriate permission.

Where practical, composite foreign keys can strengthen membership-related integrity. The service layer remains responsible for authorization.

---

## 9. Indexing strategy

Indexes are selected based on expected application queries.

| Table | Index |
|---|---|
| User | UNIQUE(email) |
| Workspace | UNIQUE(slug) |
| Membership | UNIQUE(user_id, workspace_id) |
| Membership | (workspace_id, status) |
| Membership | (user_id, status) |
| Invitation | (workspace_id, status, expires_at) |
| Invitation | Partial UNIQUE(workspace_id, normalized_email) WHERE status = PENDING |
| Project | (workspace_id, is_archived, created_at) |
| Project | (manager_id) |
| Task | (project_id, status) |
| Task | (assignee_id, status) |
| Task | (due_at) |
| Comment | (task_id, created_at) |
| ActivityLog | (workspace_id, created_at) |
| Notification | (recipient_user_id, read_at, created_at) |

Unique constraints already create supporting indexes, so duplicate indexes should not be added.

Partial indexes can be considered for frequently queried subsets, such as overdue unfinished tasks.

Actual index effectiveness should be verified using PostgreSQL EXPLAIN ANALYZE and representative data.

---

## 10. Important transactions

Some operations involve multiple records and must be atomic.

### Workspace creation
1. Create workspace.
2. Create creator's Admin membership.
3. Commit transaction.

If either operation fails, roll back both.

### Invitation acceptance
1. Validate token.
2. Validate expiry and invitation status.
3. Verify the user's email.
4. Create or reactivate membership.
5. Mark invitation as ACCEPTED.
6. Commit transaction.

### Task assignment
1. Validate project.
2. Verify active membership of the proposed assignee.
3. Check the requester's authorization.
4. Update task assignment.
5. Create ActivityLog record.
6. Optionally create Notification.
7. Commit the database changes together.

### Project archiving
1. Verify authorization.
2. Set `is_archived = TRUE`.
3. Set `archived_at`.
4. Record the activity.
5. Commit transaction.

Tasks and comments remain preserved.

---

## 11. Expected future extensions

The initial schema is intentionally limited to the MVP.

Potential future additions:

- TaskAssignee for multiple task assignees.
- MembershipHistory for complete membership lifecycle history.
- Attachment for task and project files.
- WorkspaceSettings for configurable workspace preferences.
- Labels and TaskLabel for task categorization.
- Background job records for asynchronous processing.
- RefreshToken or Session for authentication session management.
- Dedicated audit and event processing improvements.

These should be introduced only when the corresponding features are required.

---

## 12. Final database design summary

| Design area | Decision |
|---|---|
| Database | PostgreSQL |
| ORM | Prisma |
| Architecture | Modular monolith |
| Main entities | 9 |
| Primary keys | UUID |
| User-workspace relationship | Many-to-many through Membership |
| Workspace roles | Admin, Manager, Member |
| Task assignment | Zero or one assignee |
| Project deletion | Archive |
| Membership departure | Preserve record and mark LEFT |
| Comment deletion | Soft delete |
| Activity history | ActivityLog with JSONB metadata |
| Notification references | Explicit nullable foreign keys |
| Password storage | Argon2id or bcrypt hash |
| Timestamps | TIMESTAMPTZ |
| Integrity | Foreign keys, unique constraints, CHECK constraints |
| Performance | Query-driven indexes |

**Status:** Conceptual database design finalized. Next step is implementation review and conversion into PostgreSQL DDL and Prisma models.
