# Entity-Relationship Diagram — FacilityBook

The schema is created entirely through Flyway migrations `V1`–`V11` (seed data in
`V12`–`V13`). There are eleven domain tables.

## 1. Diagram

```mermaid
erDiagram
    ROLES ||--o{ USERS : "assigned to"
    ROLES ||--o{ RESOURCE_CATEGORIES : "approver for"
    DEPARTMENTS ||--o{ USERS : "employs"
    BUILDINGS ||--o{ RESOURCES : "hosts"
    RESOURCE_CATEGORIES ||--o{ RESOURCES : "classifies"
    RESOURCES ||--o{ OPERATING_HOURS : "open during"
    RESOURCES ||--o{ RESERVATIONS : "booked as"
    USERS ||--o{ RESERVATIONS : "requests"
    RESERVATIONS ||--o{ RESERVATION_APPROVALS : "decided by"
    USERS ||--o{ RESERVATION_APPROVALS : "approves"
    RESOURCES ||--o{ MAINTENANCE_RECORDS : "maintained via"
    USERS ||--o{ MAINTENANCE_RECORDS : "reports"
    USERS ||--o{ NOTIFICATIONS : "receives"
    RESERVATIONS ||--o{ NOTIFICATIONS : "about"

    ROLES {
        smallint id PK
        varchar name UK
    }
    DEPARTMENTS {
        bigint id PK
        varchar name UK
        timestamptz created_at
        timestamptz updated_at
    }
    USERS {
        bigint id PK
        varchar email UK
        varchar password_hash
        varchar full_name
        smallint role_id FK
        bigint department_id FK
        boolean active
        timestamptz created_at
        timestamptz updated_at
    }
    BUILDINGS {
        bigint id PK
        varchar name UK
        varchar address
        timestamptz created_at
        timestamptz updated_at
    }
    RESOURCE_CATEGORIES {
        bigint id PK
        varchar name UK
        varchar description
        boolean requires_approval
        smallint approver_role_id FK
        timestamptz created_at
        timestamptz updated_at
    }
    RESOURCES {
        bigint id PK
        varchar name
        text description
        bigint category_id FK
        bigint building_id FK
        varchar floor
        varchar location_detail
        integer capacity
        varchar features
        varchar status
        timestamptz created_at
        timestamptz updated_at
    }
    OPERATING_HOURS {
        bigint id PK
        bigint resource_id FK
        smallint day_of_week
        time open_time
        time close_time
    }
    RESERVATIONS {
        bigint id PK
        bigint resource_id FK
        bigint requester_id FK
        varchar purpose
        text notes
        timestamptz start_time
        timestamptz end_time
        varchar status
        boolean needs_resolution
        timestamptz cancelled_at
        varchar cancellation_reason
        timestamptz created_at
        timestamptz updated_at
    }
    RESERVATION_APPROVALS {
        bigint id PK
        bigint reservation_id FK
        bigint approver_id FK
        varchar decision
        varchar reason
        timestamptz decided_at
    }
    MAINTENANCE_RECORDS {
        bigint id PK
        bigint resource_id FK
        bigint reported_by FK
        text reason
        varchar status
        timestamptz start_time
        timestamptz end_time
        timestamptz completed_at
        timestamptz created_at
        timestamptz updated_at
    }
    NOTIFICATIONS {
        bigint id PK
        bigint user_id FK
        varchar type
        varchar title
        varchar message
        bigint reservation_id FK
        boolean is_read
        timestamptz created_at
    }
```

## 2. Tables

| Table | Purpose | Migration |
| --- | --- | --- |
| `roles` | System roles (seeded with the five roles). | `V1` |
| `departments` | Organisation departments, used for approval routing and reporting. | `V2` |
| `users` | User accounts with role and department references. | `V3` |
| `buildings` | Buildings and locations. | `V4` |
| `resource_categories` | Resource classification **and approval policy**. | `V5` |
| `resources` | Reservable rooms, equipment, vehicles, and other assets. | `V6` |
| `operating_hours` | One open/close window per resource per weekday. | `V7` |
| `reservations` | Booking time range, requester, purpose, and status. | `V8` |
| `reservation_approvals` | Approval decisions with actor, timestamp, and reason. | `V9` |
| `maintenance_records` | Damage reports, scheduled maintenance, and downtime blocks. | `V10` |
| `notifications` | In-app notifications and read status. | `V11` |

## 3. Key relational constraints

**Foreign keys**

- `users.role_id → roles.id` (required), `users.department_id → departments.id` (optional).
- `resources.category_id → resource_categories.id` (required),
  `resources.building_id → buildings.id` (optional — vehicles have no building).
- `operating_hours.resource_id → resources.id` (`ON DELETE CASCADE`).
- `reservations.resource_id → resources.id`, `reservations.requester_id → users.id`.
- `reservation_approvals.reservation_id → reservations.id` (`ON DELETE CASCADE`),
  `reservation_approvals.approver_id → users.id`.
- `maintenance_records.resource_id → resources.id`, `maintenance_records.reported_by → users.id`.
- `notifications.user_id → users.id` (`ON DELETE CASCADE`),
  `notifications.reservation_id → reservations.id` (`ON DELETE SET NULL`).

**Uniqueness**

- `roles.name`, `departments.name`, `buildings.name`, `resource_categories.name`,
  `users.email` are unique.
- `operating_hours (resource_id, day_of_week)` is unique (one window per day).

**Check constraints**

- `resources.capacity > 0` when present; `resources.status IN ('ACTIVE','INACTIVE')`.
- `resource_categories`: `requires_approval = false OR approver_role_id IS NOT NULL`.
- `operating_hours.day_of_week BETWEEN 1 AND 7`; `close_time > open_time`.
- `reservations`: `end_time > start_time`;
  `status IN ('PENDING_APPROVAL','CONFIRMED','REJECTED','CANCELLED')`.
- `reservation_approvals.decision IN ('APPROVED','REJECTED')`.
- `maintenance_records`: `end_time > start_time`;
  `status IN ('REPORTED','SCHEDULED','IN_PROGRESS','COMPLETED','CANCELLED')`.

## 4. Double-booking protection (the critical constraint)

`reservations` carries a PostgreSQL **exclusion constraint**:

```sql
CONSTRAINT no_overlapping_reservations
    EXCLUDE USING gist (
        resource_id WITH =,
        tstzrange(start_time, end_time) WITH &&
    )
    WHERE (status IN ('PENDING_APPROVAL', 'CONFIRMED'))
```

- `tstzrange(start_time, end_time)` is half-open `[start, end)`, so back-to-back
  bookings do not conflict.
- The `WHERE` clause means only active (pending or confirmed) reservations participate;
  cancelled and rejected rows never block a slot.
- `btree_gist` (created in the same migration) lets the index combine `=` on
  `resource_id` with `&&` on the range.
- The service also takes a row lock (`findByIdForUpdate`) around mutating operations,
  and `saveAndFlush` surfaces the constraint violation inside the transaction so a losing
  concurrent request returns an error instead of a false success.

This makes double-booking impossible even if two users submit at the same instant — see
`ReservationConcurrencyTest`.

## 5. How the approval policy is modelled

The approval policy lives on `resource_categories`, not on resources, so it can be
changed for a whole class of resources at once:

| Category | `requires_approval` | `approver_role_id` | Effect |
| --- | --- | --- | --- |
| Meeting Room | `false` | `null` | Auto-confirmed. |
| Training Room | `false` | `null` | Auto-confirmed. |
| Equipment | `false` | `null` | Auto-confirmed. |
| Conference Hall | `true` | `FACILITY_MANAGER` | Facility manager approves. |
| Vehicle | `true` | `DEPARTMENT_MANAGER` | Requester's department manager approves. |

A reservation whose category requires approval is created as `PENDING_APPROVAL`; the
approver role decides who may act, and ADMIN may always act.
