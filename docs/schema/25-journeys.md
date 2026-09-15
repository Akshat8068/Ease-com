# Entity: journeys

## Purpose
Automated customer lifecycle message sequences jo predefined customer timeline moments par automatically run hote hain. Har journey me multiple steps hote hain jo wait time intervals ke baad Meta-approved message templates bhejte hain. Journeys 3 families me grouped hain (`conv`, `post`, `keep`) aur 2 categories me divided hain (`utility`, `marketing`).

---

## Table Definition

```sql
CREATE TABLE journeys (
    id               UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id        UUID        NOT NULL REFERENCES tenants(id),
    name             VARCHAR(100) NOT NULL,
    family           VARCHAR(20) NOT NULL CHECK (family IN ('conv', 'post', 'keep')),
    category         VARCHAR(20) NOT NULL CHECK (category IN ('utility', 'marketing')),
    trigger_event    VARCHAR(255) NOT NULL,
    exit_condition   TEXT        NOT NULL,
    exit_n           INTEGER     NULL,
    exit_unit        VARCHAR(10) NULL CHECK (exit_unit IN ('minutes', 'hours', 'days')),
    is_active        BOOLEAN     NOT NULL DEFAULT TRUE,
    entered          INTEGER     NOT NULL DEFAULT 0,
    sent             INTEGER     NOT NULL DEFAULT 0,
    conv             INTEGER     NOT NULL DEFAULT 0,
    revenue          BIGINT      NOT NULL DEFAULT 0,
    created_by       UUID        NOT NULL REFERENCES agents(id),
    created_at       TIMESTAMP   NOT NULL DEFAULT NOW(),
    updated_by       UUID        NULL REFERENCES agents(id),
    updated_at       TIMESTAMP   NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column         | Type         | Nullable | Default           | Description                                                        |
|----------------|--------------|----------|-------------------|--------------------------------------------------------------------|
| id             | UUID         | NO       | gen_random_uuid() | Primary key                                                        |
| tenant_id      | UUID         | NO       | —                 | FK → tenants. Kis tenant ki journey hai                            |
| name           | VARCHAR(100) | NO       | —                 | Journey display name (e.g., "Abandoned cart")                      |
| family         | VARCHAR(20)  | NO       | —                 | ENUM: `conv` (Before order), `post` (After order), `keep` (Retention)|
| category       | VARCHAR(20)  | NO       | —                 | ENUM: `utility` (No opt-in needed), `marketing` (Opt-in required)  |
| trigger_event  | VARCHAR(255) | NO       | —                 | Event description / code jo journey ko trigger karta hai           |
| exit_condition | TEXT         | NO       | —                 | Human-readable exit rule jo journey stop karta hai                 |
| exit_n         | INTEGER      | YES      | NULL              | Exit duration threshold number                                     |
| exit_unit      | VARCHAR(10)  | YES      | NULL              | ENUM: `minutes` / `hours` / `days`                                 |
| is_active      | BOOLEAN      | NO       | TRUE              | Agent toggle: journey active hai ya switched off                   |
| entered        | INTEGER      | NO       | 0                 | Total customers enrolled in this journey (analytics)              |
| sent           | INTEGER      | NO       | 0                 | Total messages sent across all steps (analytics)                   |
| conv           | INTEGER      | NO       | 0                 | Total conversions / orders resulting from journey (analytics)      |
| revenue        | BIGINT       | NO       | 0                 | Total revenue influenced in smallest currency unit (analytics)     |
| created_by     | UUID         | NO       | —                 | FK → agents. Creator agent reference                               |
| created_at     | TIMESTAMP    | NO       | NOW()             | Creation timestamp                                                 |
| updated_by     | UUID         | YES      | NULL              | FK → agents. Last agent who updated configuration                 |
| updated_at     | TIMESTAMP    | NO       | NOW()             | Last update timestamp                                              |

---

## Business Rules

1. **Predefined Out-of-the-Box:** 9 predefined journeys seed automatically per tenant.
2. **Agent Customization:** Agent can toggle `is_active`, edit wait times, assign approved templates, set offer values, and adjust exit time windows.
3. **Execution Rule:** Straight line execution (`wait` $\rightarrow$ `send` $\rightarrow$ `wait` $\rightarrow$ `send` $\rightarrow$ `exit`).
4. **Instant Config Updates:** Journey timing/variable edits save **immediately** in our system without Meta re-review. (Only template text edits require Meta review).

---

## Indexes

```sql
CREATE INDEX idx_journeys_tenant ON journeys(tenant_id);
CREATE INDEX idx_journeys_tenant_active ON journeys(tenant_id, is_active);
CREATE INDEX idx_journeys_family ON journeys(tenant_id, family);
```

---

## Relationships

| Related Entity      | Type     | Via                         |
|---------------------|----------|-----------------------------|
| tenants             | Many-One | journeys.tenant_id          |
| agents              | Many-One | journeys.created_by         |
| agents              | Many-One | journeys.updated_by         |
| journey_steps       | One-Many | journey_steps.journey_id    |
| journey_enrollments | One-Many | journey_enrollments.journey_id |
| template_sends      | One-Many | template_sends.journey_id   |
