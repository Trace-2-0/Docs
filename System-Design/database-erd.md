# Database Schema & Relational Model Architecture

This document provides a structural breakdown of the PostgreSQL Multi-Tenant Database Schema, detailing table entities, foreign key relationships, cascade behaviors, and indexing strategies.

## Full Entity-Relationship Diagram (ERD)

```mermaid
erDiagram
    COMPANIES ||--o| COMPANY_SETTINGS : "1-to-1 (Cascade Delete)"
    COMPANIES ||--o| STORAGE_CONFIGS : "1-to-1 (Cascade Delete)"
    COMPANIES ||--o{ TEAMS : "1-to-Many (Cascade Delete)"
    COMPANIES ||--o{ USERS : "1-to-Many (Cascade Delete)"
    COMPANIES ||--o{ SHIFTS : "1-to-Many (Cascade Delete)"
    COMPANIES ||--o{ SCREENSHOTS : "1-to-Many (Cascade Delete)"
    COMPANIES ||--o{ AUDIT_LOGS : "1-to-Many (Cascade Delete)"

    TEAMS ||--o{ USERS : "1-to-Many (Set Null on Delete)"

    USERS ||--o{ SHIFTS : "1-to-Many (Cascade Delete)"
    USERS ||--o{ SCREENSHOTS : "1-to-Many (Cascade Delete)"
    USERS ||--o{ DAILY_APP_USAGE : "1-to-Many (Cascade Delete)"

    SHIFTS ||--o{ BREAKS : "1-to-Many (Cascade Delete)"
    SHIFTS ||--o{ IDLE_SESSIONS : "1-to-Many (Cascade Delete)"
    SHIFTS ||--o{ SCREENSHOTS : "1-to-Many (Cascade Delete)"

    COMPANIES {
        string id PK
        string slug UK
        string name
        string email UK
        string plan
        boolean isActive
        int maxEmployees
        datetime createdAt
    }

    COMPANY_SETTINGS {
        string id PK
        string companyId FK
        int expectedWorkSecs
        int screenshotIntervalSecs
        int heartbeatGraceSecs
    }

    TEAMS {
        string id PK
        string companyId FK
        string name
        int idleThresholdSecs
    }

    USERS {
        string id PK
        string companyId FK
        string teamId FK
        string email
        string role
        string agentToken UK
        boolean isActive
    }

    SHIFTS {
        string id PK
        string companyId FK
        string userId FK
        date date
        datetime startTime
        datetime endTime
        string checkoutType
        datetime lastHeartbeatAt
        int totalWorkSecs
        int totalActiveSecs
    }

    BREAKS {
        string id PK
        string companyId FK
        string shiftId FK
        datetime startTime
        datetime endTime
        int durationSecs
    }

    SCREENSHOTS {
        string id PK
        string companyId FK
        string userId FK
        string shiftId FK
        datetime capturedAt
        string storageType
        string storageKey
    }

    DAILY_APP_USAGE {
        string id PK
        string companyId FK
        string userId FK
        date date
        json appMetrics
    }

    AUDIT_LOGS {
        string id PK
        string companyId FK
        string actorId
        string action
        datetime createdAt
    }
```

## Core Relational Hierarchy

The database follows a strict 3-tier hierarchy:
`Company (Tenant Root) -> Teams (Department Grouping) -> Users (Employees/Admins)`

This hierarchical design simplifies querying and access control. Every table below `Companies` includes a `companyId` foreign key, making it trivial to scope queries and enforce multi-tenancy.

## Database Integrity & Cascade Deletes

To prevent "orphan" records and maintain data cleanliness, the Prisma schema uses explicit referential integrity actions:

* **Cascade Delete (`onDelete: Cascade`):** When a parent record is deleted, PostgreSQL automatically purges all child records in a single database transaction. For example, deleting a `Company` automatically purges its `CompanySettings`, `Teams`, `Users`, `Shifts`, `Screenshots`, and `AuditLogs`.
* **Set Null (`onDelete: SetNull`):** When a department/team is deleted, we set `User.teamId = null`. We do not delete the employee from the company just because their department was reorganized.

## Performance Indexing Strategy (`@@index`)

Querying millions of screenshots or shift records without indexes causes full table scans, resulting in severe performance degradation. Trace relies on targeted composite indexes to ensure snappy dashboard load times:

```prisma
// Fast Daily Dashboard Queries (e.g. Get today's shifts for Company X)
@@index([companyId, date])

// Fast User Activity Timeline (e.g. Get User Y's screenshots for Date Z)
@@index([userId, capturedAt])
@@index([companyId, capturedAt])

// Fast Multi-Tenant Composite Unique Key
@@unique([companyId, email]) // The same email can register under different companies
```
