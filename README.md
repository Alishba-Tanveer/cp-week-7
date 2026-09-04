# Week 7 — Assignment 3: Architecture Decision Records

## Overview

This assignment documents important architectural decisions for the Task Management Platform that will be implemented in Phase 4.

The decisions are recorded as Architecture Decision Records (ADRs). Each ADR documents the context, competing options, decision criteria, selected approach, consequences, and conditions under which the decision may need to be reconsidered.

The ADRs remain consistent with the fixed seven-table PostgreSQL domain and the REST API contract defined in the previous Week 7 assignments.

## Included

### ADR-001 — Offset Pagination

`docs/adr/ADR-001.md`

This ADR evaluates two pagination strategies:

* Offset pagination
* Cursor pagination

The decision is to use **offset pagination** for the platform's normal paginated collection endpoints.

The decision is based on:

* The existing `page` and `pageSize` API contract
* The current expected application scale
* Client simplicity
* Direct page navigation
* Compatibility with the TypeORM `skip` and `take` approach used during Week 6

The ADR also documents the disadvantages of offset pagination and defines an expiry/reconsideration condition for cases where larger datasets or high-frequency feeds make cursor pagination more appropriate.

### ADR-002 — Role-Based Authorization with Guards

`docs/adr/ADR-002.md`

This optional ADR evaluates two approaches to role-based authorization:

* Enforcing roles inside every service method
* Enforcing common role requirements using NestJS guards

The decision is to use **NestJS guards for common role-based access control**, while retaining service-level checks for resource-specific authorization and business rules such as ownership or project membership.

The ADR also documents the trade-offs and includes an expiry/reconsideration condition for substantially more complex authorization requirements.

## Architecture Context

The Task Management Platform uses:

* **NestJS** for the backend framework
* **PostgreSQL** as the relational database
* **TypeORM** for database access
* **REST API** architecture
* **API versioning** through `/api/v1`
* A fixed seven-table business domain

The fixed domain consists of:

* `users`
* `projects`
* `project_members`
* `tasks`
* `tags`
* `task_tags`
* `comments`

## Assignment Requirements Covered

### Required

* **W1 — Real architecture decision:** Offset pagination versus cursor pagination
* **W2 — Context:** Application-specific forces affecting the decision
* **C1 — Options:** At least two defensible options with honest advantages and disadvantages
* **C2 — Decision and Consequences:** Selected option, decision criterion, and accepted negative consequences
* **C3 — Cross-check:** Consistency with the system design and REST API contract

### Optional Challenge

* **X1 — Second ADR:** ADR-002 documents a different architectural decision
* **X2 — Expiry condition:** Both ADRs define measurable conditions for reconsidering their decisions

## Relationship to Previous Week 7 Assignments

The ADRs build on the architecture and API decisions documented in the previous Week 7 assignments.

ADR-001 is directly related to the pagination approach implemented and tested during Week 6 using TypeORM `skip`, `take`, and `getManyAndCount()`, while also remaining consistent with the Week 7 REST API contract's `page` and `pageSize` parameters.

ADR-002 complements the layered NestJS architecture by separating general role-based authorization from resource-specific business authorization.

The ADRs do not change the fixed database domain or introduce additional business tables.
