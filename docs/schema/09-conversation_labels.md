# Entity: conversation_labels

## Purpose
Conversations aur labels ke beech many-to-many relationship ka bridge table. Ek conversation ke multiple labels ho sakte hain, aur ek label multiple conversations mein lag sakta hai.

---

## Table Definition

```sql
CREATE TABLE conversation_labels (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID        NOT NULL REFERENCES conversations(id),
    label_id        UUID        NOT NULL REFERENCES labels(id),
    tenant_id       UUID        NOT NULL REFERENCES tenants(id),
    created_at      TIMESTAMP   NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column          | Type      | Nullable | Default           | Description                        |
|-----------------|-----------|----------|-------------------|------------------------------------|
| id              | UUID      | NO       | gen_random_uuid() | Primary key                        |
| conversation_id | UUID      | NO       | —                 | FK → conversations.id              |
| label_id        | UUID      | NO       | —                 | FK → labels.id                     |
| tenant_id       | UUID      | NO       | —                 | FK → tenants.id (quick filter)     |
| created_at      | TIMESTAMP | NO       | NOW()             | When label was applied             |

---

## Indexes

```sql
-- Conversation ke saare labels fetch karne ke liye
CREATE INDEX idx_conv_labels_conv_id ON conversation_labels(conversation_id);

-- Label ke saare conversations fetch karne ke liye (filtering)
CREATE INDEX idx_conv_labels_label_id ON conversation_labels(label_id);

-- Duplicate label application rokne ke liye
CREATE UNIQUE INDEX idx_conv_labels_unique ON conversation_labels(conversation_id, label_id);
```

---

## Business Rules

- Ek conversation pe same label do baar nahi lag sakta (UNIQUE constraint)
- Label lagana aur hatana sirf `status = 'assigned'` pe allowed hai
- `status = 'resolved'` pe labels read-only (no insert/delete)
- Label delete karne ke liye row delete hogi (hard delete — undo possible nahi)
- `tenant_id` redundant hai lekin multi-tenant queries fast karne ke liye rakha hai

---

## Example

```
conversation_id: conv-001
labels:
  ├── Order Tracking  (label_id: lbl-001)
  └── Refund Related  (label_id: lbl-003)

conversation_labels rows:
  Row 1: conv-001 | lbl-001 | tenant-001
  Row 2: conv-001 | lbl-003 | tenant-001
```

---

## Relationships

| Related Entity | Type     | Via                              |
|----------------|----------|----------------------------------|
| conversations  | Many-One | conversation_labels.conversation_id |
| labels         | Many-One | conversation_labels.label_id     |
| tenants        | Many-One | conversation_labels.tenant_id    |
