# Entity: tenants

## Purpose
CRM clients jo Ease Commerce platform use karte hain. Har tenant ek alag business hai jiske apne agents, pages, customers aur conversations hote hain.

---

## Table Definition

```sql
CREATE TABLE tenants (
    id          UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255)    NOT NULL,
    slug        VARCHAR(100)    NOT NULL UNIQUE,
    is_active   BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMP       NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column     | Type         | Nullable | Default              | Description                          |
|------------|--------------|----------|----------------------|--------------------------------------|
| id         | UUID         | NO       | gen_random_uuid()    | Primary key                          |
| name       | VARCHAR(255) | NO       | —                    | Business/company name                |
| slug       | VARCHAR(100) | NO       | —                    | URL-safe unique identifier           |
| is_active  | BOOLEAN      | NO       | TRUE                 | Soft delete / disable tenant         |
| created_at | TIMESTAMP    | NO       | NOW()                | Record creation time                 |

---

## Indexes

```sql
-- slug fast lookup ke liye (login/routing pe use hoga)
CREATE UNIQUE INDEX idx_tenants_slug ON tenants(slug);
```

---

## Business Rules

- Har tenant ka `slug` globally unique hoga — e.g. `himalayan-organics`
- `is_active = FALSE` se tenant disable hoga, delete nahi hoga (data preserve)
- Har doosri entity mein `tenant_id` FK hoga — multi-tenancy ka base hai ye table

---

## Relationships

| Related Entity       | Type      | Via            |
|----------------------|-----------|----------------|
| agents               | One-Many  | agents.tenant_id |
| pages                | One-Many  | pages.tenant_id  |
| customers            | One-Many  | customers.tenant_id |
| conversations        | One-Many  | conversations.tenant_id |
| labels               | One-Many  | labels.tenant_id |
| reasons              | One-Many  | reasons.tenant_id |
| saved_replies        | One-Many  | saved_replies.tenant_id |
