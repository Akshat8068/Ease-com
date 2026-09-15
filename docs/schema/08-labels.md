# Entity: labels

## Purpose
Ticket categories. Agent ticket pe ek label lagata hai — jaise "Order Tracking", "Refund Related" etc. Label ek main category hai jiske under multiple reasons (sub-categories) hote hain. Jab agent reason select karta hai to label automatically set ho jata hai. Har tenant ke apne custom labels hote hain jo Automation Setting mein manage hote hain. Label agent create karta hai.

---

## Table Definition

```sql
CREATE TABLE labels (
    id          UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID            NOT NULL REFERENCES tenants(id),
    name        VARCHAR(100)    NOT NULL,
    color       VARCHAR(20),
    is_active   BOOLEAN         NOT NULL DEFAULT TRUE,
    created_by  UUID            NOT NULL REFERENCES agents(id),
    created_at  TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_by  UUID            REFERENCES agents(id),
    updated_at  TIMESTAMP       NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column     | Type         | Nullable | Default           | Description                                                              |
|------------|--------------|----------|-------------------|--------------------------------------------------------------------------|
| id         | UUID         | NO       | gen_random_uuid() | Primary key                                                              |
| tenant_id  | UUID         | NO       | —                 | FK → tenants.id. Kis tenant ka label hai                                 |
| name       | VARCHAR(100) | NO       | —                 | Label display name e.g. Order Tracking, Refund Related                   |
| color      | VARCHAR(20)  | YES      | NULL              | Hex color code e.g. #4CAF50. UI mein color badge ke liye                 |
| is_active  | BOOLEAN      | NO       | TRUE              | Soft delete — inactive labels dropdowns mein nahi dikhenge               |
| created_by | UUID         | NO       | —                 | FK → agents.id. Agent jisne label create kiya                            |
| created_at | TIMESTAMP    | NO       | NOW()             | Creation time                                                            |
| updated_by | UUID         | YES      | NULL              | FK → agents.id. Last agent jisne edit kiya                               |
| updated_at | TIMESTAMP    | NO       | NOW()             | Last edit time                                                           |

---

## Label vs Reason (Category vs Sub-category)

```
Label   = Main category    (e.g. Order Issues)
Reason  = Sub-category     (e.g. Not Delivered, Wrong Item, Delayed)

One Label → Many Reasons

When agent selects a Reason on a ticket:
→ Parent Label automatically selected
→ Agent does not need to set label manually
```

---

## Flow in Chat Inbox

```
STEP 1 — Agent creates label in Automation Setting
→ name, color, tenant_id, created_by saved

STEP 2 — Agent creates reasons under this label
→ each reason has label_id FK

STEP 3 — Ticket is assigned to agent

STEP 4 — Agent opens profile panel of ticket
→ Sees Reason dropdown (list of all reasons for this tenant)
→ Selects a reason (e.g. "Not Delivered")

STEP 5 — System auto-sets label
→ finds reason.label_id
→ sets ticket.label_id = reason.label_id
→ Label field in profile panel auto-populated

STEP 6 — Resolve button ENABLES
(reason is now selected)

STEP 7 — On ticket resolve:
→ ticket.label_id and ticket.reason_id saved permanently
→ Used for reporting in Customer Voice dashboard
```

---

## Business Rules

- Label name ek tenant ke andar unique hoga
- Labels tenant-specific hain — alag tenants ke labels alag hote hain
- Ek ticket pe ONLY ONE label hoga (set via reason selection)
- Label directly set nahi hota — reason select karne par auto-set hota hai
- Labels sirf `status = 'assigned'` tickets mein edit ho sakte hain
- `status = 'resolved'` pe labels read-only ho jaate hain
- Automation Setting mein CRUD:
  - New label create (agent)
  - Existing label edit — name, color (agent)
  - Label soft delete — is_active = false (agent)

---

## Default Labels (Seeded on tenant onboard)

```
Order Tracking
Product Enquiry
Refund Related
Other
```

---

## Indexes

```sql
-- Tenant ke saare labels fetch karne ke liye
CREATE INDEX idx_labels_tenant_id ON labels(tenant_id);

-- Name uniqueness per tenant
CREATE UNIQUE INDEX idx_labels_name_tenant ON labels(name, tenant_id);
```

---

## Relationships

| Related Entity | Type     | Via                   |
|----------------|----------|-----------------------|
| tenants        | Many-One | labels.tenant_id      |
| agents         | Many-One | labels.created_by     |
| agents         | Many-One | labels.updated_by     |
| reasons        | One-Many | reasons.label_id      |
| tickets        | One-Many | tickets.label_id      |
