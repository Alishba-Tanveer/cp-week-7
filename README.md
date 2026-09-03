# Week 7 — System Design & API Architecture

## Overview

Week 7 focuses on designing the Task Management system before implementation. Building on the PostgreSQL database and TypeORM entities developed in Weeks 5 and 6, this week defines the system architecture, database relationships, REST API contract, and key architectural decisions that will guide the NestJS backend implementation in Phase 4.

The week emphasizes clear system design, consistent API conventions, scalability, security, non-functional requirements, and documenting real engineering trade-offs.

---

## Week Objectives

Concepts covered:

* System design fundamentals
* Requirements analysis
* Entity Relationship Diagrams (ERD)
* Layered application architecture
* Client → API → Service → Repository → Database flow
* REST API design
* HTTP methods and status codes
* API versioning
* Authentication and authorization
* Role-Based Access Control (RBAC)
* Pagination, filtering, and sorting
* Offset vs cursor pagination
* Caching and cache invalidation
* Read replicas and replication lag
* Consistency considerations
* N+1 problem from the API perspective
* Architecture trade-offs
* Architecture Decision Records (ADR)
* Maintaining consistency between design documents

---

## Task Management Domain

Week 7 uses the same fixed Task Management domain implemented during Weeks 5 and 6.

The system contains seven database tables:

1. `users`
2. `projects`
3. `project_members`
4. `tasks`
5. `tags`
6. `task_tags`
7. `comments`

### Entity Relationships

* A `User` can own multiple projects.
* A `User` can be a member of multiple projects.
* A `Project` belongs to one owner.
* A `Project` can have many members.
* A `Project` can contain many tasks.
* A `ProjectMember` connects users and projects and stores the user's role.
* A `Task` belongs to one project.
* A `Task` can optionally be assigned to a user.
* A `Task` can have multiple tags.
* A `Tag` can belong to multiple tasks.
* `task_tags` implements the many-to-many relationship between tasks and tags.
* A `Task` can have multiple comments.
* A `Comment` belongs to one task and is written by one user.

### Project Member Roles

The `project_members.role` field supports:

* `owner`
* `admin`
* `member`
* `viewer`

These roles will be used for Role-Based Access Control in the Phase 4 backend.

---

## Week 7 Assignments

Week 7 contains three main graded assignments.

### Assignment 1 — System Design Document

Deliverable: `DESIGN.md`

The system design document describes how the Task Management platform is structured and how its components interact.

It includes:

* System overview
* Functional requirements
* Non-functional requirements
* ERD
* Layered architecture
* Create Task request flow
* Validation and authorization responsibilities
* Error handling
* Caching strategy
* Scaling strategy
* Consistency considerations
* Engineering trade-offs

The ERD must represent the fixed seven-table schema and maintain the same entity and column names used in the previous database work.

#### Required Architecture Layers

The design follows a layered architecture:

Client
   ↓
API / Controller
   ↓
Service
   ↓
Repository
   ↓
PostgreSQL Database

Each layer has a defined responsibility and should avoid taking over responsibilities belonging to another layer.

---

### Assignment 2 — REST API Contract

Deliverable: `api-spec.md`

The API contract defines the REST API that will be implemented during the NestJS backend phase.

The specification covers:

* Tasks
* Users
* Projects
* Project members
* Tags
* Comments
* Authentication

### API Versioning

All API routes use:

/api/v1

Example:

/api/v1/tasks

### Task Endpoints

The tasks resource includes:

* `POST /api/v1/tasks`
* `GET /api/v1/tasks`
* `GET /api/v1/tasks/:id`
* `PATCH /api/v1/tasks/:id`
* `DELETE /api/v1/tasks/:id`

### Nested Resources

Resources that belong to another resource use nested routes where appropriate.

Examples:

/api/v1/projects/:id/members
/api/v1/tasks/:id/comments

### Pagination Convention

List endpoints follow a common pagination and filtering convention:

?page=&pageSize=&status=&sort=

The API specification defines:

* Default page
* Default page size
* Maximum page size
* Filtering
* Sorting
* Paged response structure
* Total record count

### Authentication

The API contract includes:

* Register
* Login
* Refresh token
* Logout

Authentication endpoints define their request bodies, response bodies, and relevant HTTP status codes.

Sensitive fields such as `password_hash` are never returned in API responses.

### Error Contract

API errors use one consistent structure:

{
  statusCode,
  message,
  error,
  timestamp,
  path
}

The API contract provides examples for:

* `400 Bad Request`
* `401 Unauthorized`
* `403 Forbidden`
* `404 Not Found`

Additional conflict handling includes:

* `409 Conflict`

for situations such as duplicate emails or duplicate tag names.

### Access Levels

Routes are classified as:

* Public
* Authenticated
* Role-restricted

Role-restricted endpoints specify the required project roles.

---

## Assignment 3 — Architecture Decision Record

Deliverable: `ADR-001.md`

The Architecture Decision Record documents one important technical decision made during the system design process.

The ADR follows the structure:

1. Context
2. Options Considered
3. Decision
4. Consequences

The purpose is to explain not only what decision was made, but also why it was selected and what trade-offs are accepted.

Potential architectural decisions include:

* Offset pagination vs cursor pagination
* Role enforcement in guards vs services
* Caching vs read replicas

---

## Non-Functional Design

The Week 7 system design considers several non-functional concerns.

### Performance

List endpoints use pagination to avoid returning unnecessarily large datasets.

A maximum `pageSize` is defined to protect the API from excessively large requests.

### Caching

A project task list can be considered for caching to reduce repeated database reads.

When a task-changing operation affects the cached project task list, the corresponding cache entry must be invalidated.

Caching introduces the cost of possible stale data and cache invalidation complexity.

### Scaling

The system can use a primary database for writes and read replicas for suitable read operations.

                ┌──→ Read Replica
API → Repository ┤
                └──→ Primary Database
                       ↑
                     Writes

The trade-off is replication lag, meaning a recent write may not immediately appear on a replica.

Operations requiring the latest committed data should use the primary database.

### Consistency

The design considers where strong consistency is required and where eventual consistency is acceptable.

Examples of operations that may require the primary database include:

* Permission checks
* Immediately-after-write reads
* Critical authorization decisions

---

## Engineering Trade-Offs

Week 7 emphasizes that every architectural choice has a cost.

Examples of decisions considered include:

### Offset Pagination vs Cursor Pagination

**Offset pagination**

Advantages:

* Simple to implement
* Easy to understand
* Supports direct page navigation

Disadvantages:

* Can become slower for deep pages
* Results can shift when records are inserted or deleted

**Cursor pagination**

Advantages:

* Better for large datasets
* More stable for continuously changing data
* Efficient for deep traversal

Disadvantages:

* More complex
* Directly jumping to an arbitrary page is difficult

The decision is based on the expected requirements and complexity of the Task Management application.

### Normalization vs Denormalization

Normalized data reduces duplication and keeps writes simpler.

Denormalization can improve read performance but introduces additional write complexity and consistency concerns.

### Guard-Based vs Service-Based Authorization

Centralized authorization guards can keep controllers and services cleaner and provide consistent access control.

Service-level checks can provide more context-specific control but may result in duplicated authorization logic if not carefully structured.

---

## Repository Structure

The Week 7 repository contains the design deliverables:

week-7/
├── README.md
├── DESIGN.md
├── api-spec.md
└── ADR-001.md

Optional challenge work add:

├── ADR-002.md