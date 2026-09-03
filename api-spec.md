# Task Management Platform — REST API Contract

## Week 7 — Assignment 2

This document defines the REST API contract that the Phase 4 NestJS backend will implement for the fixed Task Management Platform domain.

The business domain contains these seven tables:

* `users`
* `projects`
* `project_members`
* `tasks`
* `tags`
* `task_tags`
* `comments`

Phase 4 authentication may add authentication-specific storage such as `password_hash` on `users` and a `refresh_tokens` table. These authentication details do not change the seven-table business domain.

All application routes use the `/api/v1` version prefix.

Authentication uses JWT access tokens. Authorization is based on project-level membership roles:

* `owner`
* `admin`
* `member`
* `viewer`

There is no global user-role column. Project roles are scoped to individual projects.

All timestamps use ISO 8601 UTC date-time values. Date-only values such as `dueDate` use `YYYY-MM-DD`.

---

# W1 — Tasks Resource

A task belongs to exactly one project and may optionally be assigned to one user.

Task status values are:

* `todo`
* `in_progress`
* `done`

Priority is an integer from `1` to `5`.

## Task Response Representation

```json
{
  "id": 1,
  "title": "Implement task filtering",
  "description": "Add status-based filtering",
  "status": "todo",
  "priority": 3,
  "projectId": 1,
  "assigneeId": 2,
  "dueDate": "2026-09-10",
  "createdAt": "2026-09-03T10:00:00Z",
  "tags": [
    {
      "id": 1,
      "name": "backend"
    }
  ]
}
```

`description`, `assigneeId`, and `dueDate` may be `null`.

---

## W1.1 — Create Task

### Request

```http
POST /api/v1/tasks
Authorization: Bearer <access-token>
```

### Access

Authenticated users with one of these roles in the target project may create tasks:

* `owner`
* `admin`
* `member`

`viewer` cannot create tasks.

### Request Body

```json
{
  "title": "Implement task filtering",
  "description": "Add status-based filtering",
  "status": "todo",
  "priority": 3,
  "projectId": 1,
  "assigneeId": 2,
  "dueDate": "2026-09-10",
  "tagIds": [1, 3]
}
```

### Fields

| Field         | Type         | Required | Rules                           |
| ------------- | ------------ | -------: | ------------------------------- |
| `title`       | string       |      Yes | 1–255 characters                |
| `description` | string/null  |       No | Maximum 5000 characters         |
| `status`      | enum         |      Yes | `todo`, `in_progress`, `done`   |
| `priority`    | integer      |      Yes | 1–5                             |
| `projectId`   | integer      |      Yes | Existing project                |
| `assigneeId`  | integer/null |       No | Existing user or null           |
| `dueDate`     | date/null    |       No | `YYYY-MM-DD` or null            |
| `tagIds`      | integer[]    |       No | Existing tag IDs; unique values |

If `tagIds` is omitted or empty, the task has no tag associations.

The project must exist and the authenticated user must be a member of it.

If `assigneeId` is supplied, the user must exist.

If `tagIds` are supplied, every tag must exist.

### Success

**201 Created**

Returns the created `Task`.

### Errors

* `400 Bad Request` — invalid body or validation failure
* `401 Unauthorized` — missing/invalid authentication
* `403 Forbidden` — user lacks task-creation permission
* `404 Not Found` — project, assignee, or referenced tag does not exist
* `409 Conflict` — conflicting relationship/database state

---

## W1.2 — List Tasks

### Request

```http
GET /api/v1/tasks?page=1&pageSize=20&status=todo&sort=-createdAt
Authorization: Bearer <access-token>
```

### Access

Authenticated users may only receive tasks belonging to projects in which they are members.

The server applies authorization before returning task records.

### Filters

Supported filters:

* `projectId`
* `assigneeId`
* `tagId`
* `status`

If `projectId` is supplied, it must identify a project accessible to the authenticated user. An inaccessible project is treated as outside the user's resource scope.

### Success

**200 OK**

```json
{
  "data": [],
  "total": 0,
  "page": 1,
  "pageSize": 20,
  "totalPages": 0,
  "hasNext": false,
  "hasPrevious": false
}
```

### Errors

* `400 Bad Request` — invalid query parameter
* `401 Unauthorized` — user is not authenticated

---

## W1.3 — Get One Task

### Request

```http
GET /api/v1/tasks/:id
Authorization: Bearer <access-token>
```

### Access

Authenticated users must be members of the task's project.

### Success

**200 OK**

Returns the `Task` representation.

### Errors

* `401 Unauthorized`
* `403 Forbidden` — resource existence may safely be revealed but user lacks access
* `404 Not Found` — task does not exist or its existence is intentionally hidden

---

## W1.4 — Update Task

### Request

```http
PATCH /api/v1/tasks/:id
Authorization: Bearer <access-token>
```

PATCH is a partial update. Only supplied fields are changed.

### Access

Permitted project roles:

* `owner`
* `admin`
* `member`

`viewer` cannot update tasks.

### Updatable Fields

* `title`
* `description`
* `status`
* `priority`
* `assigneeId`
* `dueDate`
* `tagIds`

`id`, `projectId`, and `createdAt` cannot be changed.

Keeping `projectId` immutable prevents an update from silently moving a task between authorization scopes.

### Tag Replacement Semantics

If `tagIds` is omitted, existing task-tag relationships remain unchanged.

If `tagIds` is supplied, it completely replaces the existing task-tag set.

An empty `tagIds` array removes all tags from the task.

### Success

**200 OK**

Returns the updated `Task`.

### Errors

* `400 Bad Request`
* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`
* `409 Conflict`

---

## W1.5 — Delete Task

### Request

```http
DELETE /api/v1/tasks/:id
Authorization: Bearer <access-token>
```

### Access

Permitted roles:

* `owner`
* `admin`

### Delete Behavior

Deleting a task cascades to:

* its `task_tags` rows
* its `comments`

This matches the fixed database relationship behavior.

### Success

**204 No Content**

No response body.

### Errors

* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`

---

# W2 — Pagination, Filtering and Sorting

All list endpoints use the same pagination convention.

## W2.1 Pagination

### `page`

* Type: integer
* Default: `1`
* Minimum: `1`

### `pageSize`

* Type: integer
* Default: `20`
* Minimum: `1`
* Maximum: `100`

Requests with `pageSize > 100` return `400 Bad Request`.

### Offset

The backend uses:

```text
offset = (page - 1) * pageSize
```

This is compatible with the Week 6 TypeORM `skip`, `take`, and `getManyAndCount()` approach.

### Paged Response

Every paginated list returns:

```json
{
  "data": [],
  "total": 0,
  "page": 1,
  "pageSize": 20,
  "totalPages": 0,
  "hasNext": false,
  "hasPrevious": false
}
```

Rules:

```text
totalPages = ceil(total / pageSize)
hasNext = page < totalPages
hasPrevious = page > 1 && total > 0
```

For an empty result, `totalPages` is `0` and both navigation flags are `false`.

---

## W2.2 Sorting

The `sort` parameter identifies a whitelisted field.

A leading `-` means descending order.

Examples:

```text
sort=createdAt
sort=-createdAt
```

The backend always adds `id ASC` as the final tie-breaker unless `id` itself is already the requested field.

### Supported Task Sorting

* `id`
* `-id`
* `createdAt`
* `-createdAt`
* `title`
* `-title`
* `priority`
* `-priority`
* `dueDate`
* `-dueDate`
* `status`
* `-status`

Default:

```text
sort=-createdAt
```

### Supported User Sorting

* `id`
* `-id`
* `name`
* `-name`
* `email`
* `-email`
* `createdAt`
* `-createdAt`

Default: `id`

### Supported Project Sorting

* `id`
* `-id`
* `name`
* `-name`
* `createdAt`
* `-createdAt`

Default: `id`

### Supported Project Member Sorting

* `userId`
* `-userId`
* `role`
* `-role`

Default: `userId`

### Supported Tag Sorting

* `id`
* `-id`
* `name`
* `-name`

Default: `name`

### Supported Comment Sorting

* `id`
* `-id`
* `createdAt`
* `-createdAt`

Default: `createdAt`

Invalid sort values return `400 Bad Request`.

---

# C1 — Remaining Resources

The remaining resources are:

1. Users
2. Projects
3. Project Members
4. Tags
5. Comments

Nested resources are used where the relationship is fundamental to the resource.

---

# C1.1 — Users

User creation is performed through registration:

```http
POST /api/v1/auth/register
```

There is intentionally no separate `POST /api/v1/users` endpoint.

## List Users

```http
GET /api/v1/users
Authorization: Bearer <access-token>
```

Authenticated users may access basic user profiles.

Response: **200 OK**

Uses `PagedUsers`.

Errors:

* `400 Bad Request`
* `401 Unauthorized`

## Get User

```http
GET /api/v1/users/:id
Authorization: Bearer <access-token>
```

Response: **200 OK**

Returns:

```json
{
  "id": 1,
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "createdAt": "2026-09-03T10:00:00Z"
}
```

Errors:

* `401 Unauthorized`
* `404 Not Found`

## Update Own User

```http
PATCH /api/v1/users/:id
Authorization: Bearer <access-token>
```

Only the authenticated user's own profile may be changed.

API v1 permits changing `name`.

`email` is immutable through this endpoint.

Request:

```json
{
  "name": "Alice Smith"
}
```

Response: **200 OK**

Errors:

* `400 Bad Request`
* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`

## Delete Own User

```http
DELETE /api/v1/users/:id
Authorization: Bearer <access-token>
```

Only the authenticated user's own account may be deleted.

Project memberships are removed according to the defined relationship behavior.

A user who owns a project cannot be deleted until project ownership is resolved.

A user referenced by required comment authorship cannot be deleted while those comments remain.

If a restrictive relationship prevents deletion:

**409 Conflict**

Success:

**204 No Content**

Errors:

* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`
* `409 Conflict`

Authentication secrets are never returned.

---

# C1.2 — Projects

## Create Project

```http
POST /api/v1/projects
Authorization: Bearer <access-token>
```

Authenticated users may create projects.

The authenticated user becomes the project owner.

Creation must atomically establish:

```text
projects.ownerId = authenticated user ID
```

and:

```text
project_members:
(userId = authenticated user ID,
 projectId = new project ID,
 role = owner)
```

These two representations must remain consistent.

Request:

```json
{
  "name": "Website Redesign"
}
```

Response: **201 Created**

```json
{
  "id": 1,
  "name": "Website Redesign",
  "ownerId": 1,
  "createdAt": "2026-09-03T10:00:00Z"
}
```

Errors:

* `400 Bad Request`
* `401 Unauthorized`

A `409` is not defined for project-name uniqueness because `projects.name` is not unique in the fixed schema.

## List Projects

```http
GET /api/v1/projects
Authorization: Bearer <access-token>
```

Returns only projects for which the authenticated user has membership.

Response: **200 OK**

Uses `PagedProjects`.

Errors:

* `400 Bad Request`
* `401 Unauthorized`

## Get Project

```http
GET /api/v1/projects/:id
Authorization: Bearer <access-token>
```

Authenticated project members may access project details.

Errors:

* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`

## Update Project

```http
PATCH /api/v1/projects/:id
Authorization: Bearer <access-token>
```

Permitted roles:

* `owner`
* `admin`

Only `name` may be changed.

`ownerId` cannot be changed through this endpoint.

Ownership transfer requires a separate explicit workflow and is outside API v1.

Response: **200 OK**

Errors:

* `400 Bad Request`
* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`

## Delete Project

```http
DELETE /api/v1/projects/:id
Authorization: Bearer <access-token>
```

Permitted roles:

* `owner`
* `admin`

Deletion cascades to:

* project memberships
* project tasks

Task deletion then cascades to:

* task-tag associations
* task comments

This follows the required fixed-domain relationship behavior.

Response: **204 No Content**

Errors:

* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`

---

# C1.3 — Project Members

Project members are nested under projects.

## Add Project Member

```http
POST /api/v1/projects/:id/members
Authorization: Bearer <access-token>
```

Permitted roles:

* `owner`
* `admin`

Request:

```json
{
  "userId": 2,
  "role": "member"
}
```

Allowed assigned roles:

* `admin`
* `member`
* `viewer`

The `owner` role cannot be assigned through this endpoint.

The project owner membership is created during project creation.

Response: **201 Created**

Errors:

* `400 Bad Request`
* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`
* `409 Conflict`

`409` covers an existing `(userId, projectId)` membership.

## List Project Members

```http
GET /api/v1/projects/:id/members
Authorization: Bearer <access-token>
```

Any project member may list members.

The response includes the owner.

Response: **200 OK**

Uses `PagedProjectMembers`.

Errors:

* `400 Bad Request`
* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`

## Update Member Role

```http
PATCH /api/v1/projects/:id/members/:userId
Authorization: Bearer <access-token>
```

Permitted roles:

* `owner`
* `admin`

Allowed target roles:

* `admin`
* `member`
* `viewer`

The owner cannot be demoted through ordinary membership management because `projects.ownerId` and the owner membership must remain consistent.

Response: **200 OK**

Errors:

* `400 Bad Request`
* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`
* `409 Conflict`

## Remove Member

```http
DELETE /api/v1/projects/:id/members/:userId
Authorization: Bearer <access-token>
```

Permitted roles:

* `owner`
* `admin`

The canonical project owner cannot be removed.

Ownership transfer is outside API v1.

Response: **204 No Content**

Errors:

* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`
* `409 Conflict`

---

# C1.4 — Tags

Tags are global because the fixed schema does not associate tags directly with projects.

Any authenticated user may create, update, or delete global tags. This is an intentional API policy because the fixed domain has no global administrator role.

## Create Tag

```http
POST /api/v1/tags
Authorization: Bearer <access-token>
```

Request:

```json
{
  "name": "frontend"
}
```

Response: **201 Created**

Errors:

* `400 Bad Request`
* `401 Unauthorized`
* `409 Conflict`

`409` is returned when the unique tag name already exists.

## List Tags

```http
GET /api/v1/tags
Authorization: Bearer <access-token>
```

Response: **200 OK**

Uses `PagedTags`.

Errors:

* `400 Bad Request`
* `401 Unauthorized`

## Get Tag

```http
GET /api/v1/tags/:id
Authorization: Bearer <access-token>
```

Response: **200 OK**

Errors:

* `401 Unauthorized`
* `404 Not Found`

## Update Tag

```http
PATCH /api/v1/tags/:id
Authorization: Bearer <access-token>
```

Request:

```json
{
  "name": "frontend-ui"
}
```

Response: **200 OK**

Errors:

* `400 Bad Request`
* `401 Unauthorized`
* `404 Not Found`
* `409 Conflict`

## Delete Tag

```http
DELETE /api/v1/tags/:id
Authorization: Bearer <access-token>
```

The tag itself is not automatically deleted from `task_tags`.

Because the fixed relationship is non-cascading from `task_tags.tag_id` to `tags.id`, a tag still referenced by tasks cannot be deleted.

The client must remove all associated task-tag relationships first.

Response:

**204 No Content**

Errors:

* `401 Unauthorized`
* `404 Not Found`
* `409 Conflict`

---

# C1.5 — Comments

Comments belong to exactly one task and are nested under tasks.

## Create Comment

```http
POST /api/v1/tasks/:taskId/comments
Authorization: Bearer <access-token>
```

Any project member may comment, including `viewer`.

The authenticated user becomes `authorId`.

`authorId` is never accepted from the client.

Request:

```json
{
  "body": "The filtering implementation is ready for review."
}
```

Response: **201 Created**

```json
{
  "id": 1,
  "taskId": 1,
  "authorId": 2,
  "body": "The filtering implementation is ready for review.",
  "createdAt": "2026-09-03T10:00:00Z"
}
```

Errors:

* `400 Bad Request`
* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`

## List Comments

```http
GET /api/v1/tasks/:taskId/comments
Authorization: Bearer <access-token>
```

Any member of the task's project may list comments.

Response: **200 OK**

Uses `PagedComments`.

Errors:

* `400 Bad Request`
* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`

## Get Comment

```http
GET /api/v1/tasks/:taskId/comments/:commentId
Authorization: Bearer <access-token>
```

Any member of the task's project may retrieve the comment.

Errors:

* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`

## Update Comment

```http
PATCH /api/v1/tasks/:taskId/comments/:commentId
Authorization: Bearer <access-token>
```

Only the comment author may update the comment.

Request:

```json
{
  "body": "Updated comment text."
}
```

Response: **200 OK**

Errors:

* `400 Bad Request`
* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`

## Delete Comment

```http
DELETE /api/v1/tasks/:taskId/comments/:commentId
Authorization: Bearer <access-token>
```

A comment may be deleted by:

* its author
* the project owner
* a project admin

Response: **204 No Content**

Errors:

* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`

---

# C2 — Authentication

Authentication endpoints are under:

```text
/api/v1/auth
```

Phase 4 authentication adds the necessary authentication storage, including a password hash and refresh-token persistence. The password hash is never returned by any API response.

## C2.1 Register

```http
POST /api/v1/auth/register
```

Public endpoint.

Request:

```json
{
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "password": "SecurePassword123!"
}
```

Password requirements:

* minimum 8 characters
* maximum 128 characters

Response:

**201 Created**

```json
{
  "user": {
    "id": 1,
    "name": "Alice Johnson",
    "email": "alice@example.com",
    "createdAt": "2026-09-03T10:00:00Z"
  },
  "accessToken": "access-token-value",
  "refreshToken": "refresh-token-value"
}
```

Errors:

* `400 Bad Request`
* `409 Conflict` — email already exists

## C2.2 Login

```http
POST /api/v1/auth/login
```

Public endpoint.

Request:

```json
{
  "email": "alice@example.com",
  "password": "SecurePassword123!"
}
```

Response:

**200 OK**

Returns the user, access token, and refresh token.

Invalid credentials return:

**401 Unauthorized**

The API does not reveal whether the supplied email exists.

## C2.3 Refresh

```http
POST /api/v1/auth/refresh
```

The endpoint is public but requires a valid refresh token.

Request:

```json
{
  "refreshToken": "refresh-token-value"
}
```

Response:

**200 OK**

```json
{
  "accessToken": "new-access-token-value",
  "refreshToken": "new-refresh-token-value"
}
```

Refresh-token rotation is required.

The previous refresh token is revoked after successful rotation.

Errors:

* `400 Bad Request`
* `401 Unauthorized`

## C2.4 Logout

```http
POST /api/v1/auth/logout
Authorization: Bearer <access-token>
```

Request:

```json
{
  "refreshToken": "refresh-token-value"
}
```

The supplied refresh token is revoked.

Response:

**204 No Content**

Errors:

* `400 Bad Request`
* `401 Unauthorized`

---

# C3 — Error Contract

All API errors use the same top-level structure:

```json
{
  "statusCode": 400,
  "message": "Priority must be an integer between 1 and 5",
  "error": "Bad Request",
  "timestamp": "2026-09-03T10:05:00.000Z",
  "path": "/api/v1/tasks"
}
```

The `message` field is always a string.

## 400 Bad Request

Used for malformed requests, validation errors, invalid pagination, invalid filters, and invalid sorting.

## 401 Unauthorized

Used when authentication is missing, invalid, expired, revoked, or unsuccessful.

## 403 Forbidden

Used when the authenticated user is known but lacks the required permission.

## 404 Not Found

Used when the requested resource does not exist.

For private resources, `404` may also be deliberately returned when revealing resource existence would leak information.

## 409 Conflict

Used for:

* duplicate unique values
* duplicate project membership
* duplicate task-tag association
* relationship restrictions
* ownership restrictions
* conflicting resource state
* idempotency-key reuse with a different payload

---

# C4 — Versioning and Access Control

All application routes use:

```text
/api/v1
```

Future incompatible API changes may be introduced under a new version such as `/api/v2`.

## Public

* `POST /api/v1/auth/register`
* `POST /api/v1/auth/login`
* `POST /api/v1/auth/refresh` with valid refresh token

## Authenticated

A valid JWT access token is required for protected operations.

## Project Authorization

| Role   | Read | Create Task | Update Task | Delete Task | Manage Members | Update/Delete Project |
| ------ | ---: | ----------: | ----------: | ----------: | -------------: | --------------------: |
| owner  |  Yes |         Yes |         Yes |         Yes |            Yes |                   Yes |
| admin  |  Yes |         Yes |         Yes |         Yes |            Yes |                   Yes |
| member |  Yes |         Yes |         Yes |          No |             No |                    No |
| viewer |  Yes |          No |          No |          No |             No |                    No |

Additional resource rules:

* All project members may create comments.
* Only comment authors may update comments.
* Comment authors, project owners, and project admins may delete comments.
* Any authenticated user may manage global tags.
* Users may update/delete only their own account.
* The server always derives authorization from authoritative membership data.

---

# C4.1 — Complete Route Access Summary

| Method | Route                                       | Access                        |
| ------ | ------------------------------------------- | ----------------------------- |
| POST   | `/api/v1/auth/register`                     | Public                        |
| POST   | `/api/v1/auth/login`                        | Public                        |
| POST   | `/api/v1/auth/refresh`                      | Public + valid refresh token  |
| POST   | `/api/v1/auth/logout`                       | Authenticated                 |
| GET    | `/api/v1/users`                             | Authenticated                 |
| GET    | `/api/v1/users/:id`                         | Authenticated                 |
| PATCH  | `/api/v1/users/:id`                         | Own account                   |
| DELETE | `/api/v1/users/:id`                         | Own account                   |
| POST   | `/api/v1/projects`                          | Authenticated                 |
| GET    | `/api/v1/projects`                          | Authenticated                 |
| GET    | `/api/v1/projects/:id`                      | Project member                |
| PATCH  | `/api/v1/projects/:id`                      | Owner/Admin                   |
| DELETE | `/api/v1/projects/:id`                      | Owner/Admin                   |
| POST   | `/api/v1/projects/:id/members`              | Owner/Admin                   |
| GET    | `/api/v1/projects/:id/members`              | Project member                |
| PATCH  | `/api/v1/projects/:id/members/:userId`      | Owner/Admin                   |
| DELETE | `/api/v1/projects/:id/members/:userId`      | Owner/Admin                   |
| POST   | `/api/v1/tasks`                             | Owner/Admin/Member            |
| GET    | `/api/v1/tasks`                             | Authenticated + project scope |
| GET    | `/api/v1/tasks/:id`                         | Project member                |
| PATCH  | `/api/v1/tasks/:id`                         | Owner/Admin/Member            |
| DELETE | `/api/v1/tasks/:id`                         | Owner/Admin                   |
| GET    | `/api/v1/tasks/:taskId/tags`                | Project member                |
| POST   | `/api/v1/tasks/:taskId/tags`                | Owner/Admin/Member            |
| DELETE | `/api/v1/tasks/:taskId/tags/:tagId`         | Owner/Admin/Member            |
| POST   | `/api/v1/tags`                              | Authenticated                 |
| GET    | `/api/v1/tags`                              | Authenticated                 |
| GET    | `/api/v1/tags/:id`                          | Authenticated                 |
| PATCH  | `/api/v1/tags/:id`                          | Authenticated                 |
| DELETE | `/api/v1/tags/:id`                          | Authenticated                 |
| POST   | `/api/v1/tasks/:taskId/comments`            | Project member                |
| GET    | `/api/v1/tasks/:taskId/comments`            | Project member                |
| GET    | `/api/v1/tasks/:taskId/comments/:commentId` | Project member                |
| PATCH  | `/api/v1/tasks/:taskId/comments/:commentId` | Comment author                |
| DELETE | `/api/v1/tasks/:taskId/comments/:commentId` | Author/Owner/Admin            |

---

# X1 — Idempotent Task Creation

Task creation supports the optional:

```http
Idempotency-Key: <client-generated-key>
```

Example:

```http
POST /api/v1/tasks
Authorization: Bearer <access-token>
Idempotency-Key: abc123
```

The key is scoped to:

* authenticated user
* HTTP operation
* route

When a new key is received:

1. The request is processed.
2. The task is created.
3. The successful response is stored with the idempotency record.
4. A retry using the same key returns the original successful result.
5. No duplicate task is created.

If the same key is reused with a different request payload:

**409 Conflict**

The idempotency record must be persisted in a durable mechanism.

Recommended retention:

**24 hours**

The exact persistence implementation may be finalized during Phase 4, but the externally observable retry behavior is part of this contract.

---

# X2 — Task Status Codes

| Status             | Meaning                                                         |
| ------------------ | --------------------------------------------------------------- |
| `200 OK`           | Successful task retrieval/update                                |
| `201 Created`      | Successful task creation                                        |
| `204 No Content`   | Successful task deletion                                        |
| `400 Bad Request`  | Invalid body, query, status, priority, pagination, or parameter |
| `401 Unauthorized` | Missing or invalid authentication                               |
| `403 Forbidden`    | Authenticated user lacks permission                             |
| `404 Not Found`    | Resource does not exist or is intentionally hidden              |
| `409 Conflict`     | Duplicate or conflicting state                                  |

## X2.1 — 403 vs 404

`403 Forbidden` means the resource is known to exist and the caller is authenticated but lacks permission.

`404 Not Found` means the resource does not exist.

For private project/task resources, the backend may intentionally use `404` when returning `403` would reveal that a protected resource exists.

The implementation must apply this policy consistently.

---

# X3 — OpenAPI 3.1

A machine-readable OpenAPI 3.1 specification is provided separately as:

```text
openapi.yaml
```

The OpenAPI specification represents this same contract and contains:

* all `/api/v1` routes
* HTTP methods
* path parameters
* query parameters
* request bodies
* response bodies
* success status codes
* error status codes
* authentication requirements
* project authorization requirements
* JWT bearer security scheme
* pagination schemas
* resource schemas
* authentication schemas
* common error schema
* `Idempotency-Key`
* reusable `$ref` components

`api-spec.md` and `openapi.yaml` must remain synchronized.

---

# Final Design Principles

1. Every application route uses `/api/v1`.
2. Protected routes require authentication.
3. Project authorization is determined server-side from project membership.
4. The four project roles are `owner`, `admin`, `member`, and `viewer`.
5. `PATCH` is used for partial updates.
6. Successful creation returns `201 Created`.
7. Successful deletion returns `204 No Content`.
8. Invalid input returns `400 Bad Request`.
9. Authentication failures return `401 Unauthorized`.
10. Authorization failures return `403 Forbidden`.
11. Missing or intentionally hidden private resources return `404 Not Found`.
12. Conflicting state returns `409 Conflict`.
13. List endpoints use 1-based pagination.
14. Default `pageSize` is `20`.
15. Maximum `pageSize` is `100`.
16. Pagination uses `offset = (page - 1) * pageSize`.
17. List sorting is whitelist-based.
18. `id ASC` is used as the final deterministic tie-breaker.
19. Project/task data is never exposed outside the user's authorized project scope.
20. Task creation supports optional idempotency.
21. Refresh tokens are rotated.
22. Passwords and password hashes are never returned.
23. Nested project-member routes are under projects.
24. Nested comments are under tasks.
25. Task-tag relationships are exposed through nested task routes.
26. Project deletion cascades to project memberships and tasks.
27. Task deletion cascades to task-tag relationships and comments.
28. Global tag deletion is blocked while restrictive task-tag references remain.
29. Project ownership and owner membership must remain consistent.
30. The OpenAPI specification and written contract must describe the same API.
