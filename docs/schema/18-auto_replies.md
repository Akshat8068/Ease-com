# Entity: auto_replies

## Purpose
Har tenant ke liye predefined automatic messages jo new ticket create hone par user ko automatically bheje jaate hain — bina kisi agent action ke. Teen fixed types hain jo dev team ne define kiye hain. Agent sirf message text, channel aur on/off toggle edit kar sakta hai.

---

## Table Definition

```sql
CREATE TABLE auto_replies (
    id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID        NOT NULL REFERENCES tenants(id),
    type         VARCHAR(20) NOT NULL CHECK (type IN ('instant', 'away', 'sla_breach')),
    message      TEXT        NOT NULL,
    channel      VARCHAR(20) NOT NULL CHECK (channel IN ('instagram', 'facebook', 'whatsapp', 'all')),
    is_active    BOOLEAN     NOT NULL DEFAULT TRUE,
    updated_by   UUID        REFERENCES agents(id),
    updated_at   TIMESTAMP   NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column     | Type        | Nullable | Default           | Description                                                                 |
|------------|-------------|----------|-------------------|-----------------------------------------------------------------------------|
| id         | UUID        | NO       | gen_random_uuid() | Primary key                                                                 |
| tenant_id  | UUID        | NO       | —                 | FK → tenants. Kis tenant ka auto reply hai                                  |
| type       | VARCHAR(20) | NO       | —                 | ENUM: instant / away / sla_breach. Predefined by dev, agent cannot change   |
| message    | TEXT        | NO       | —                 | Message text jo user ko bheja jayega. Agent edit kar sakta hai              |
| channel    | VARCHAR(20) | NO       | —                 | ENUM: instagram / facebook / whatsapp / all. Agent select karta hai         |
| is_active  | BOOLEAN     | NO       | TRUE              | Agent toggle kar sakta hai on/off                                           |
| updated_by | UUID        | YES      | NULL              | FK → agents. Last agent jisne edit kiya                                     |
| updated_at | TIMESTAMP   | NO       | NOW()             | Last edit ka time                                                           |

---

## Type Definitions (Predefined — Dev Team)

| Type        | When it fires                                      | Purpose                                           |
|-------------|----------------------------------------------------|---------------------------------------------------|
| instant     | Ticket arrives WITHIN working hours                | User ko batao agent jald reply karega             |
| away        | Ticket arrives OUTSIDE working hours               | User ko batao team offline hai, working hours batao |
| sla_breach  | SLA breach hone wali hai ya ho gayi               | User ko batao delay ho raha hai                   |

---

## Business Rules

- Teen types FIXED hain — agent naye type create nahi kar sakta, existing delete nahi kar sakta
- Har tenant ke liye teen rows seed hoti hain (one per type) jab tenant onboard hota hai
- Agent sirf `message`, `channel`, aur `is_active` edit kar sakta hai
- `type` kabhi update nahi hota — dev team define karta hai
- Working hours ke saath runtime par decide hota hai kaun sa type fire karega
- `channel = all` matlab har channel par fire karega jab ticket aaye

---

## How it connects to working_hours at runtime

```
Ticket arrives
→ Check working_hours (tenant_id + current day + current time)

IN working hours:
→ Find auto_reply WHERE tenant_id = ? AND type = 'instant' AND is_active = true AND (channel = ticket.channel OR channel = 'all')
→ Send that message

OUT of working hours:
→ Find auto_reply WHERE tenant_id = ? AND type = 'away' AND is_active = true AND (channel = ticket.channel OR channel = 'all')
→ Send that message
```

---

## Indexes

```sql
-- Tenant ke saare auto replies fast fetch karne ke liye
CREATE INDEX idx_auto_replies_tenant ON auto_replies(tenant_id);

-- Runtime check: tenant + type + channel + is_active
CREATE INDEX idx_auto_replies_tenant_type_channel ON auto_replies(tenant_id, type, channel, is_active);
```

---

## Relationships

| Related Entity | Type      | Via                        |
|----------------|-----------|----------------------------|
| tenants        | Many-One  | auto_replies.tenant_id     |
| agents         | Many-One  | auto_replies.updated_by    |
| working_hours  | Runtime   | tenant_id (no FK, runtime) |
