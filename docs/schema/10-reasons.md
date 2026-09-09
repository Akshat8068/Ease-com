# Entity: reasons

## Purpose
Conversation resolve karne ka internal business reason. Resolve karne se pehle agent ko ek reason select karna mandatory hai. Reasons department-wise grouped hote hain. Har tenant ke apne custom reasons hote hain jo Automation Setting mein manage hote hain.

---

## Table Definition

```sql
CREATE TABLE reasons (
    id          UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID            NOT NULL REFERENCES tenants(id),
    name        VARCHAR(255)    NOT NULL,
    department  VARCHAR(100)    NOT NULL,
    description TEXT,
    is_active   BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMP       NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column      | Type         | Nullable | Default           | Description                                       |
|-------------|--------------|----------|-------------------|---------------------------------------------------|
| id          | UUID         | NO       | gen_random_uuid() | Primary key                                       |
| tenant_id   | UUID         | NO       | —                 | FK → tenants.id                                   |
| name        | VARCHAR(255) | NO       | —                 | Reason display name                               |
| department  | VARCHAR(100) | NO       | —                 | Operational owner/team                            |
| description | TEXT         | YES      | NULL              | Additional context (shown below reason dropdown)  |
| is_active   | BOOLEAN      | NO       | TRUE              | Inactive reasons naye conversations mein nahi dikhenge |
| created_at  | TIMESTAMP    | NO       | NOW()             | Creation time                                     |

---

## Allowed Values

### department
| Value     | Description                   |
|-----------|-------------------------------|
| logistics | Delivery, courier, AWB issues |
| warehouse | Packing, dispatch issues      |
| finance   | Refund, payment issues        |
| inventory | Stock, availability issues    |
| catalog   | Product info, listing issues  |
| platform  | App/website technical issues  |
| ops       | General operations            |
| marketing | Campaign, offer related       |

---

## Example Reasons

```
LOGISTICS
├── Delivery running late
├── Wrong address delivery attempted
└── Courier not reachable

FINANCE
├── Refund processing
├── Bank side delay
└── Refund completed

WAREHOUSE
├── Wrong item shipped
└── Damaged product dispatched
```

---

## Indexes

```sql
-- Tenant ke saare reasons fetch karne ke liye
CREATE INDEX idx_reasons_tenant_id ON reasons(tenant_id);

-- Department wise grouped display ke liye
CREATE INDEX idx_reasons_tenant_dept ON reasons(tenant_id, department);

-- Name uniqueness per tenant
CREATE UNIQUE INDEX idx_reasons_name_tenant ON reasons(name, tenant_id);
```

---

## Business Rules

- Reason name ek tenant ke andar unique hoga
- Reasons tenant-specific hain
- Conversation resolve karne se pehle `conversations.reason_id` set hona zaroori hai
- `status = 'resolved'` ke baad reason read-only ho jaata hai
- Automation Setting mein CRUD:
  - New reason create (name + department mandatory)
  - Existing reason edit
  - Reason delete (`is_active = FALSE`)
- `description` field profile panel mein reason ke niche show hoti hai (jaise "Counted under Logistics on Customer Voice...")

---

## Relationships

| Related Entity | Type     | Via                       |
|----------------|----------|---------------------------|
| tenants        | Many-One | reasons.tenant_id         |
| conversations  | One-Many | conversations.reason_id   |
