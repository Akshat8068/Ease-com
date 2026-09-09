# Entity: assignment_history

## Purpose
Har conversation ke assignment ka poora trail. Kab aaya, kisne assign kiya, kisse kisko assign hua, kab resolve hua, kab reopen hua — sab kuch yahan record hota hai. History icon se agents ye trail dekh sakte hain.

---

## Table Definition

```sql
CREATE TABLE assignment_history (
    id              UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID            NOT NULL REFERENCES conversations(id),
    tenant_id       UUID            NOT NULL REFERENCES tenants(id),
    assigned_from   UUID            REFERENCES agents(id),
    assigned_to     UUID            REFERENCES agents(id),
    assigned_by     UUID            REFERENCES agents(id),
    action          VARCHAR(50)     NOT NULL,
    note            TEXT,
    created_at      TIMESTAMP       NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column          | Type        | Nullable | Default           | Description                                        |
|-----------------|-------------|----------|-------------------|----------------------------------------------------|
| id              | UUID        | NO       | gen_random_uuid() | Primary key                                        |
| conversation_id | UUID        | NO       | —                 | FK → conversations.id                              |
| tenant_id       | UUID        | NO       | —                 | FK → tenants.id                                    |
| assigned_from   | UUID        | YES      | NULL              | FK → agents.id (pehle kiske paas thi, NULL = unassigned) |
| assigned_to     | UUID        | YES      | NULL              | FK → agents.id (ab kiske paas gayi, NULL = unassigned)   |
| assigned_by     | UUID        | YES      | NULL              | FK → agents.id (kisne assign kiya, NULL = system)  |
| action          | VARCHAR(50) | NO       | —                 | Kya hua                                            |
| note            | TEXT        | YES      | NULL              | Additional context                                 |
| created_at      | TIMESTAMP   | NO       | NOW()             | Event time                                         |

---

## Allowed Values

### action
| Value         | assigned_from | assigned_to | Description                                    |
|---------------|---------------|-------------|------------------------------------------------|
| created       | NULL          | NULL        | Thread arrived — conversation bani             |
| assigned      | NULL          | Agent       | Pehli baar assign hua                          |
| reassigned    | Agent A       | Agent B     | Ek agent se doosre ko transfer                 |
| self_assigned | NULL          | Agent       | Agent ne khud le li                            |
| resolved      | Agent         | NULL        | Agent ne resolve kiya                          |
| auto_resolved | Agent         | NULL        | System ne auto-resolve kiya (timer expire)     |
| reopened      | NULL          | Agent/NULL  | Reopen kiya — assignee tha to uske paas wapas  |

---

## Indexes

```sql
-- Conversation ki poori history chronologically
CREATE INDEX idx_assign_history_conv ON assignment_history(conversation_id, created_at);

-- Agent wise history (kaun kaun si tickets handle ki)
CREATE INDEX idx_assign_history_tenant_agent ON assignment_history(tenant_id, assigned_to);
```

---

## Business Rules

- Har assignment change pe ek nayi row insert hogi (append-only table)
- Rows kabhi delete ya update nahi hongi — audit trail hai
- `assigned_by = NULL` tab hoga jab system ne auto action kiya
- `assigned_from = NULL` tab hoga jab pehli baar assign ho rahi ho
- Profile panel mein "Assign History" section yahi data show karta hai

---

## Example Trail

```
1. created       | from: NULL  | to: NULL   | "Thread arrived from Instagram"
2. assigned      | from: NULL  | to: Rahul  | "Assigned by Sandeep"
3. reassigned    | from: Rahul | to: Priya  | "Reassigned by Rahul"
4. resolved      | from: Priya | to: NULL   | "Resolved by Priya"
5. reopened      | from: NULL  | to: Priya  | "Reopened — returned to assignee"
```

---

## Relationships

| Related Entity | Type     | Via                               |
|----------------|----------|-----------------------------------|
| conversations  | Many-One | assignment_history.conversation_id|
| tenants        | Many-One | assignment_history.tenant_id      |
| agents         | Many-One | assignment_history.assigned_from  |
| agents         | Many-One | assignment_history.assigned_to    |
| agents         | Many-One | assignment_history.assigned_by    |
