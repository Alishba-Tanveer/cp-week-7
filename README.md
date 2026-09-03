**# Week 7 — Assignment 2: REST API Contract**

**## Overview**

This assignment defines the REST API contract for the Task Management Platform that will be implemented in Phase 4.

The API contract is documented in `api-spec.md` and `openapi.yaml`, based on the fixed seven-table PostgreSQL domain defined in Assignment 1.

**## Included**

* Complete `tasks` resource with:

* Create task

* List tasks

* Get one task

* Update task

* Delete task

* Common pagination, filtering, and sorting conventions

* Users and projects endpoints

* Nested project member endpoints

* Tags and task-tag operations

* Nested task comment endpoints

* Authentication endpoints:

* Register

* Login

* Refresh

* Logout

* Standard error response contract

* API versioning using `/api/v1`

* Authentication and role-based authorization requirements

* HTTP status codes and REST semantics

* Optional idempotency design for task creation

* Status-code and security considerations

* OpenAPI 3.1 specification in `openapi.yaml`
