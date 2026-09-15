# Entity: working_hours

## Purpose
Har tenant ke liye business working hours define karta hai — day by day. Ye table do kaam karta hai:
1. Control karta hai kaun sa auto reply fire karega (instant ya away)
2. SLA clock ka start/pause/resume decide karta hai

---

## Table Definition

```sql
CREATE TABLE working_hours (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID        NOT NULL REFERENCES tenants(id),
    day         VARCHAR(10) NOT NULL CHECK (day IN ('monday', 'tuesday', 'wednesday', 'thursday', 'friday', 'saturday', 'sunday')),
    start_time  TIME        NOT NULL,
    end_time    TIME        NOT NULL,
    is_active   BOOLEAN     NOT NULL DEFAULT TRUE,
    updated_by  UUID        REFERENCES agents(id),
    updated_at  TIMESTAMP   NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column     | Type        | Nullable | Default           | Description                                                              |
|------------|-------------|----------|-------------------|--------------------------------------------------------------------------|
| id         | UUID        | NO       | gen_random_uuid() | Primary key                                                              |
| tenant_id  | UUID        | NO       | —                 | FK → tenants. Kis tenant ki working hours hain                           |
| day        | VARCHAR(10) | NO       | —                 | ENUM: monday to sunday. Ek row per day per tenant                        |
| start_time | TIME        | NO       | —                 | Business start time e.g. 09:00. Agent edit kar sakta hai                 |
| end_time   | TIME        | NO       | —                 | Business end time e.g. 18:00. Agent edit kar sakta hai                   |
| is_active  | BOOLEAN     | NO       | TRUE              | Is din business open hai ya nahi. Agent toggle kar sakta hai             |
| updated_by | UUID        | YES      | NULL              | FK → agents. Last agent jisne edit kiya                                  |
| updated_at | TIMESTAMP   | NO       | NOW()             | Last edit ka time                                                        |

---

## Business Rules

- Har tenant ke liye exactly 7 rows hoti hain (monday se sunday)
- Ye 7 rows SEED hoti hain jab tenant onboard hota hai — agent create nahi karta
- Agent sirf `start_time`, `end_time`, aur `is_active` edit kar sakta hai
- `day` column kabhi change nahi hota
- No foreign key to auto_replies or sla_configs — runtime par tenant_id se connect hota hai

---

## Default Seed (on tenant onboard)

```sql
INSERT INTO working_hours (tenant_id, day, start_time, end_time, is_active)
VALUES
(new_tenant_id, 'monday',    '09:00', '18:00', true),
(new_tenant_id, 'tuesday',   '09:00', '18:00', true),
(new_tenant_id, 'wednesday', '09:00', '18:00', true),
(new_tenant_id, 'thursday',  '09:00', '18:00', true),
(new_tenant_id, 'friday',    '09:00', '18:00', true),
(new_tenant_id, 'saturday',  '00:00', '00:00', false),
(new_tenant_id, 'sunday',    '00:00', '00:00', false);
```

---

## How it connects at runtime

```
Ticket arrives
→ system reads working_hours
  WHERE tenant_id = this tenant
    AND day       = current day
    AND is_active = true

→ Check: current time BETWEEN start_time AND end_time

IN hours:
→ instant auto reply fires
→ SLA clock starts immediately

OUT of hours (or is_active = false):
→ away auto reply fires
→ SLA clock starts at next working slot start
```

---

## 24hr Tenant Case

```
If tenant works 24 hours:
→ All 7 days active
→ start_time = 00:00
→ end_time   = 23:59
→ SLA clock always starts immediately
→ instant auto reply always fires
→ away auto reply never fires
```

---

## Indexes

```sql
-- Tenant ke saari working hours fast fetch karne ke liye
CREATE INDEX idx_working_hours_tenant ON working_hours(tenant_id);

-- Runtime check: tenant + day + is_active
CREATE UNIQUE INDEX idx_working_hours_tenant_day ON working_hours(tenant_id, day);
```

---

## Relationships

| Related Entity | Type     | Via                          |
|----------------|----------|------------------------------|
| tenants        | Many-One | working_hours.tenant_id      |
| agents         | Many-One | working_hours.updated_by     |
| auto_replies   | Runtime  | tenant_id (no FK, runtime)   |
| sla_configs    | Runtime  | tenant_id (no FK, runtime)   |
