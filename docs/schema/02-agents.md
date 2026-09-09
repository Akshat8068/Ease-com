# Entity: agents

## Purpose
Tenant ke employees / customer executives jo conversations handle karte hain. Ye log Chat Inbox mein login karke tickets assign karte hain, reply karte hain aur resolve karte hain.

---

## Table Definition

```sql
CREATE TABLE agents (
    id          UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID            NOT NULL REFERENCES tenants(id),
    name        VARCHAR(255)    NOT NULL,
    email       VARCHAR(255)    NOT NULL,
    avatar_url  TEXT,
    role        VARCHAR(50)     NOT NULL DEFAULT 'agent',
    is_active   BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP
);
```

---

## Columns

| Column     | Type         | Nullable | Default           | Description                              |
|------------|--------------|----------|-------------------|------------------------------------------|
| id         | UUID         | NO       | gen_random_uuid() | Primary key                              |
| tenant_id  | UUID         | NO       | —                 | FK → tenants.id                          |
| name       | VARCHAR(255) | NO       | —                 | Agent ka full name                       |
| email      | VARCHAR(255) | NO       | —                 | Login email (tenant ke andar unique)     |
| avatar_url | TEXT         | YES      | NULL              | Profile picture URL                      |
| role       | VARCHAR(50)  | NO       | 'agent'           | 'agent' ya 'admin'                       |
| is_active  | BOOLEAN      | NO       | TRUE              | Inactive agents ko conversations nahi milti |
| created_at | TIMESTAMP    | NO       | NOW()             | Record creation time                     |
| updated_at | TIMESTAMP    | YES      | NULL              | Last update time                         |

---

## Allowed Values

### role
| Value   | Description                        |
|---------|------------------------------------|
| agent   | Regular customer support agent     |
| admin   | Tenant admin — full access         |

---

## Indexes

```sql
-- Tenant wise agents fetch karne ke liye
CREATE INDEX idx_agents_tenant_id ON agents(tenant_id);

-- Email login aur uniqueness check ke liye
CREATE UNIQUE INDEX idx_agents_email_tenant ON agents(email, tenant_id);
```

---

## Business Rules

- Email ek tenant ke andar unique hoga (alag tenants mein same email allowed)
- `is_active = FALSE` — soft delete, data preserve hoga
- Saare agents ko New Messages ki saari unassigned tickets dikhti hain (koi senior restriction nahi)
- Koi bhi agent kisi bhi agent ko assign kar sakta hai (khud ko bhi)
- `role = 'admin'` ke liye future mein permission layer add ho sakti hai

---

## Relationships

| Related Entity       | Type      | Via                              |
|----------------------|-----------|----------------------------------|
| tenants              | Many-One  | agents.tenant_id                 |
| conversations        | One-Many  | conversations.assigned_to        |
| conversations        | One-Many  | conversations.resolved_by        |
| assignment_history   | One-Many  | assignment_history.assigned_to   |
| assignment_history   | One-Many  | assignment_history.assigned_from |
| assignment_history   | One-Many  | assignment_history.assigned_by   |
| saved_replies        | One-Many  | saved_replies.created_by         |
