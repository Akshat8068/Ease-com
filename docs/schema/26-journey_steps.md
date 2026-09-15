# Entity: journey_steps

## Purpose
Journey steps define each individual message dispatch point inside a journey sequence. Every step specifies how long to wait after the trigger (or previous step), which Meta-approved template to send, dynamic offer parameters (e.g. `10%` discount), and optional step execution conditions (e.g. `COD only`).

---

## Table Definition

```sql
CREATE TABLE journey_steps (
    id               UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    journey_id       UUID        NOT NULL REFERENCES journeys(id) ON DELETE CASCADE,
    step_order       INTEGER     NOT NULL,
    wait_n           INTEGER     NULL,
    wait_unit        VARCHAR(10) NULL CHECK (wait_unit IN ('minutes', 'hours', 'days')),
    wait_label       VARCHAR(50) NOT NULL,
    template_id      UUID        NOT NULL REFERENCES templates(id),
    condition        VARCHAR(50) NULL,
    offer_value      VARCHAR(20) NULL,
    note             TEXT        NULL,
    created_at       TIMESTAMP   NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMP   NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_journey_step_order UNIQUE (journey_id, step_order)
);
```

---

## Columns

| Column      | Type        | Nullable | Default           | Description                                                        |
|-------------|-------------|----------|-------------------|--------------------------------------------------------------------|
| id          | UUID        | NO       | gen_random_uuid() | Primary key                                                        |
| journey_id  | UUID        | NO       | —                 | FK → journeys. Parent journey reference                            |
| step_order  | INTEGER     | NO       | —                 | Step sequence order (1, 2, 3...)                                   |
| wait_n      | INTEGER     | YES      | NULL              | Delay number (e.g., 45, 24, 0 for immediate)                       |
| wait_unit   | VARCHAR(10) | YES      | NULL              | ENUM: `minutes` / `hours` / `days`                                 |
| wait_label  | VARCHAR(50) | NO       | —                 | Human-readable label (e.g., "45 minutes", "+ 24 hours", "Day 25")  |
| template_id | UUID        | NO       | —                 | FK → templates. Must point to an approved Meta template            |
| condition   | VARCHAR(50) | YES      | NULL              | Optional condition check (e.g. "COD only", "if rated 4 or 5")       |
| offer_value | VARCHAR(20) | YES      | NULL              | Dynamic value injected into template variable {{Offer}} (e.g. 10%) |
| note        | TEXT        | YES      | NULL              | Operational note explaining delay and messaging strategy           |
| created_at  | TIMESTAMP   | NO       | NOW()             | Creation timestamp                                                 |
| updated_at  | TIMESTAMP   | NO       | NOW()             | Last update timestamp                                              |

---

## Business Rules

1. **Approved Template Check:** A step can only fire if `template_id` points to a template with `status = 'Approved'` at send time.
2. **Dynamic Variables:** `offer_value` allows changing discount amounts directly in journey settings without resubmitting the template to Meta.
3. **Conditional Steps:** Steps with a `condition` (e.g. `COD only` in Order Confirmation) evaluate at runtime before sending.

---

## Indexes

```sql
CREATE INDEX idx_journey_steps_journey ON journey_steps(journey_id);
CREATE INDEX idx_journey_steps_template ON journey_steps(template_id);
```

---

## Relationships

| Related Entity | Type     | Via                         |
|----------------|----------|-----------------------------|
| journeys       | Many-One | journey_steps.journey_id    |
| templates      | Many-One | journey_steps.template_id   |
| template_sends | One-Many | template_sends.journey_step_id |
