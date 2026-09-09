# Entity: labels

## Purpose
Conversation categories. Agent conversation pe multiple labels laga sakta hai — jaise "Order Tracking", "Refund Related" etc. Har tenant ke apne custom labels hote hain jo Automation Setting mein manage hote hain.

---

## Table Definition

```sql
CREATE TABLE labels (
    id          UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID            NOT NULL REFERENCES tenants(id),
    name        VARCHAR(100)    NOT NULL,
    color       VARCHAR(20),
    is_active   BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMP       NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column     | Type         | Nullable | Default           | Description                              |
|------------|--------------|----------|-------------------|------------------------------------------|
| id         | UUID         | NO       | gen_random_uuid() | Primary key                              |
| tenant_id  | UUID         | NO       | —                 | FK → tenants.id                          |
| name       | VARCHAR(100) | NO       | —                 | Label display name                       |
| color      | VARCHAR(20)  | YES      | NULL              | Hex color code (e.g. #4CAF50)            |
| is_active  | BOOLEAN      | NO       | TRUE              | Inactive labels naye conversations mein nahi dikhenge |
| created_at | TIMESTAMP    | NO       | NOW()             | Creation time                            |

---

## Default Labels (Per Tenant)

```
Order Tracking
Product Enquiry
Refund Related
Other
```

Ye defaults tenant create hone ke time insert kiye jaate hain. Tenant inhe edit/delete bhi kar sakta hai.

---

## Indexes

```sql
-- Tenant ke saare labels fetch karne ke liye
CREATE INDEX idx_labels_tenant_id ON labels(tenant_id);

-- Name uniqueness per tenant
CREATE UNIQUE INDEX idx_labels_name_tenant ON labels(name, tenant_id);
```

---

## Business Rules

- Label name ek tenant ke andar unique hoga
- Labels tenant-specific hain — alag tenants ke labels alag hote hain
- Conversation pe multiple labels lag sakte hain (many-to-many via conversation_labels)
- Labels sirf `status = 'assigned'` conversations mein edit ho sakte hain
- `status = 'resolved'` pe labels read-only ho jaate hain
- Automation Setting mein CRUD operations:
  - New label create
  - Existing label edit (name, color)
  - Label delete (`is_active = FALSE`)

---

## Relationships

| Related Entity      | Type     | Via                         |
|---------------------|----------|-----------------------------|
| tenants             | Many-One | labels.tenant_id            |
| conversation_labels | One-Many | conversation_labels.label_id|
