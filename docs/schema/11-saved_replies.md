# Entity: saved_replies

## Purpose
Pre-defined message templates jo agents chat composer se quick replies ke liye use karte hain. Merge fields support karte hain (customer name, order ID, AWB etc.) aur conditional clauses bhi hote hain jo field missing hone pe automatically drop ho jaate hain.

---

## Table Definition

```sql
CREATE TABLE saved_replies (
    id          UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID            NOT NULL REFERENCES tenants(id),
    name        VARCHAR(255)    NOT NULL,
    content     TEXT            NOT NULL,
    is_active   BOOLEAN         NOT NULL DEFAULT TRUE,
    created_by  UUID            REFERENCES agents(id),
    created_at  TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP
);
```

---

## Columns

| Column     | Type         | Nullable | Default           | Description                                          |
|------------|--------------|----------|-------------------|------------------------------------------------------|
| id         | UUID         | NO       | gen_random_uuid() | Primary key                                          |
| tenant_id  | UUID         | NO       | —                 | FK → tenants.id                                      |
| name       | VARCHAR(255) | NO       | —                 | Template display name (e.g. "Share tracking")        |
| content    | TEXT         | NO       | —                 | Template body with merge fields and conditionals     |
| is_active  | BOOLEAN      | NO       | TRUE              | Inactive templates composer mein nahi dikhenge       |
| created_by | UUID         | YES      | NULL              | FK → agents.id (kisne banaya)                        |
| created_at | TIMESTAMP    | NO       | NOW()             | Creation time                                        |
| updated_at | TIMESTAMP    | YES      | NULL              | Last edit time                                       |

---

## Merge Fields

| Field        | Replaced With              | If Missing         |
|--------------|----------------------------|--------------------|
| `{{name}}`   | Customer name              | "there"            |
| `{{order}}`  | Linked order ID            | "your order"       |
| `{{courier}}`| Courier name               | "our courier partner" |
| `{{awb}}`    | AWB tracking number        | clause dropped     |
| `{{agent}}`  | Agent's own name           | clause dropped     |

---

## Conditional Clause Syntax

```
[[ text here ]]
```

Agar is clause mein koi required field missing hai to poora clause drop ho jaata hai.

---

## Example Templates

### Share Tracking
```
Hi {{name}}, order {{order}} is on its way with {{courier}} 
— AWB {{awb}}. You can track it any time, and the courier 
will call before delivery.
```

Without linked order renders as:
```
Hi Aarav, your order is on its way with our courier partner. 
You can track it any time, and the courier will call before delivery.
```

### Apologise for delay
```
Hi {{name}}, I am sorry order {{order}} has not reached you 
yet. I have raised it with the courier and will come back 
to you within 24 hours with an update.
```

### Refund timeline
```
Hi {{name}}, the refund for order {{order}} has been approved. 
It reaches the original payment method in 3–5 working days.
```

---

## Indexes

```sql
-- Tenant ke saare saved replies fetch karne ke liye
CREATE INDEX idx_saved_replies_tenant_id ON saved_replies(tenant_id);

-- Name search in composer dropdown
CREATE INDEX idx_saved_replies_name_tenant ON saved_replies(name, tenant_id);
```

---

## Business Rules

- Saved replies tenant-specific hain — saare agents ek tenant ke sare templates use kar sakte hain
- Meta approval ki zaroorat nahi (free-form messages hain)
- Automation Setting mein CRUD:
  - New saved reply create
  - Existing edit (name + content)
  - Delete (`is_active = FALSE`)
- Composer mein "Saved Replies" dropdown se search aur select hota hai
- Select karne ke baad content composer mein insert hota hai — agent send karne se pehle edit kar sakta hai
- Merge fields server-side render honge conversation context se (linked order, customer name etc.)

---

## Relationships

| Related Entity | Type     | Via                        |
|----------------|----------|----------------------------|
| tenants        | Many-One | saved_replies.tenant_id    |
| agents         | Many-One | saved_replies.created_by   |
