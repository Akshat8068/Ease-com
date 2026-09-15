# Entity: journey_enrollments

## Purpose
Tracks live customer progress through a journey. When a client API trigger occurs (e.g. cart creation or order placement), an enrollment record is created. The background journey execution worker uses this table to schedule next step dispatches and to immediately terminate dispatches when exit conditions are met (e.g. when order is completed).

---

## Table Definition

```sql
CREATE TABLE journey_enrollments (
    id               UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    journey_id       UUID        NOT NULL REFERENCES journeys(id),
    tenant_id        UUID        NOT NULL REFERENCES tenants(id),
    customer_id      UUID        NOT NULL REFERENCES customers(id),
    order_id         VARCHAR(100) NULL,
    status           VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'completed', 'exited')),
    current_step     INTEGER     NOT NULL DEFAULT 1,
    enrolled_at      TIMESTAMP   NOT NULL DEFAULT NOW(),
    exited_at        TIMESTAMP   NULL,
    exit_reason      VARCHAR(100) NULL,
    created_at       TIMESTAMP   NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMP   NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column       | Type         | Nullable | Default           | Description                                                        |
|--------------|--------------|----------|-------------------|--------------------------------------------------------------------|
| id           | UUID         | NO       | gen_random_uuid() | Primary key                                                        |
| journey_id   | UUID         | NO       | —                 | FK → journeys. Target journey                                      |
| tenant_id    | UUID         | NO       | —                 | FK → tenants. Tenant scoping                                       |
| customer_id  | UUID         | NO       | —                 | FK → customers. Enrolled customer reference                        |
| order_id     | VARCHAR(100) | YES      | NULL              | Client API reference (Order ID, Cart ID, or Shipment AWB)           |
| status       | VARCHAR(20)  | NO       | 'active'          | ENUM: `active` (In progress), `completed` (Done), `exited` (Stopped)|
| current_step | INTEGER      | NO       | 1                 | Sequence order number of next/current step                         |
| enrolled_at  | TIMESTAMP    | NO       | NOW()             | Enrollment start time                                              |
| exited_at    | TIMESTAMP    | YES      | NULL              | Exit / Completion timestamp                                        |
| exit_reason  | VARCHAR(100) | YES      | NULL              | Reason string e.g. "order placed", "timeout 72h", "delivered"       |
| created_at   | TIMESTAMP    | NO       | NOW()             | Record creation timestamp                                          |
| updated_at   | TIMESTAMP    | NO       | NOW()             | Record last update timestamp                                       |

---

## Business Rules

1. **Sends Once Rule:** An enrollment prevents duplicate runs for the same `(journey_id, customer_id, order_id)` combination.
2. **Exit Check:** Prior to executing any step `wait`, the system checks whether an exit event has occurred. If so, `status` changes to `'exited'`, `exited_at` is set to `NOW()`, and execution terminates.
3. **Completion:** When all steps are sent, `status` changes to `'completed'`.

---

## Indexes

```sql
CREATE INDEX idx_journey_enrollments_active ON journey_enrollments(journey_id, status) WHERE status = 'active';
CREATE INDEX idx_journey_enrollments_customer ON journey_enrollments(tenant_id, customer_id);
CREATE INDEX idx_journey_enrollments_order ON journey_enrollments(tenant_id, order_id);
```

---

## Relationships

| Related Entity | Type     | Via                                  |
|----------------|----------|--------------------------------------|
| journeys       | Many-One | journey_enrollments.journey_id       |
| tenants        | Many-One | journey_enrollments.tenant_id        |
| customers      | Many-One | journey_enrollments.customer_id      |
