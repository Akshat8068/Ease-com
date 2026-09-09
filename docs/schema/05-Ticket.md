# Entity: conversations

## Purpose
Core entity — har ek ticket/conversation ka record. Ek conversation ek customer + channel + page ka combination hai. Jab tak conversation resolve nahi hoti, naye messages isi mein add hote rehte hain. Resolve hone ke baad agar customer phir message kare to nayi conversation banti hai.

---

## Table Definition

```sql
CREATE TABLE conversations (
    id                      UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID            NOT NULL REFERENCES tenants(id),
    customer_id             UUID            NOT NULL REFERENCES customers(id),
    page_id                 UUID            NOT NULL REFERENCES pages(id),
    channel                 VARCHAR(50)     NOT NULL,
    source_type             VARCHAR(50)     NOT NULL DEFAULT 'direct_message',
    external_thread_id      VARCHAR(255),
    ticket_number           VARCHAR(50)     NOT NULL UNIQUE,
    status                  VARCHAR(20)     NOT NULL DEFAULT 'new',
    priority                VARCHAR(20)     NOT NULL DEFAULT 'normal',
    assigned_to             UUID            REFERENCES agents(id),
    linked_order_id         VARCHAR(255),
    linked_order_data       JSONB,
    about                   TEXT,
    reason_id               UUID            REFERENCES reasons(id),
    is_helpful              BOOLEAN,
    csat_rating             SMALLINT,
    csat_rated_at           TIMESTAMP,
    auto_resolved           BOOLEAN         NOT NULL DEFAULT FALSE,
    resolved_at             TIMESTAMP,
    resolved_by             UUID            REFERENCES agents(id),
    first_message_at        TIMESTAMP,
    last_message_at         TIMESTAMP,
    sla_first_response_due  TIMESTAMP,
    sla_resolution_due      TIMESTAMP,
    sla_first_response_met  BOOLEAN         NOT NULL DEFAULT FALSE,
    sla_resolution_met      BOOLEAN         NOT NULL DEFAULT FALSE,
    created_at              TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMP
);
```

---

## Columns

| Column                  | Type         | Nullable | Default           | Description                                          |
|-------------------------|--------------|----------|-------------------|------------------------------------------------------|
| id                      | UUID         | NO       | gen_random_uuid() | Primary key                                          |
| tenant_id               | UUID         | NO       | —                 | FK → tenants.id                                      |
| customer_id             | UUID         | NO       | —                 | FK → customers.id                                    |
| page_id                 | UUID         | NO       | —                 | FK → pages.id (konse page pe aaya)                   |
| channel                 | VARCHAR(50)  | NO       | —                 | 'instagram' / 'facebook' / 'whatsapp'                |
| source_type             | VARCHAR(50)  | NO       | 'direct_message'  | Conversation ka origin                               |
| external_thread_id      | VARCHAR(255) | YES      | NULL              | Integration service ka thread/conversation ID        |
| ticket_number           | VARCHAR(50)  | NO       | —                 | Display ticket number (e.g. "Ticket 11")             |
| status                  | VARCHAR(20)  | NO       | 'new'             | Conversation ka current state                        |
| priority                | VARCHAR(20)  | NO       | 'normal'          | SLA priority level                                   |
| assigned_to             | UUID         | YES      | NULL              | FK → agents.id (currently assigned agent)            |
| linked_order_id         | VARCHAR(255) | YES      | NULL              | Client ke order system ka Order ID                   |
| linked_order_data       | JSONB        | YES      | NULL              | Order snapshot {amount, platform, date, order_number}|
| about                   | TEXT         | YES      | NULL              | Agent ka internal note about this conversation       |
| reason_id               | UUID         | YES      | NULL              | FK → reasons.id (mandatory before resolve)           |
| is_helpful              | BOOLEAN      | YES      | NULL              | CSAT: TRUE=helpful, FALSE=not helpful, NULL=no reply |
| csat_rating             | SMALLINT     | YES      | NULL              | 1-5 star rating from customer                        |
| csat_rated_at           | TIMESTAMP    | YES      | NULL              | When customer gave rating                            |
| auto_resolved           | BOOLEAN      | NO       | FALSE             | TRUE if auto-resolved due to no CSAT response        |
| resolved_at             | TIMESTAMP    | YES      | NULL              | When conversation was resolved                       |
| resolved_by             | UUID         | YES      | NULL              | FK → agents.id (who resolved)                        |
| first_message_at        | TIMESTAMP    | YES      | NULL              | Customer ka pehla message time                       |
| last_message_at         | TIMESTAMP    | YES      | NULL              | Last message time (list ordering ke liye)            |
| sla_first_response_due  | TIMESTAMP    | YES      | NULL              | First response deadline                              |
| sla_resolution_due      | TIMESTAMP    | YES      | NULL              | Resolution deadline                                  |
| sla_first_response_met  | BOOLEAN      | NO       | FALSE             | First response SLA met?                              |
| sla_resolution_met      | BOOLEAN      | NO       | FALSE             | Resolution SLA met?                                  |
| created_at              | TIMESTAMP    | NO       | NOW()             | Ticket creation time                                 |
| updated_at              | TIMESTAMP    | YES      | NULL              | Last update                                          |

---

## Allowed Values

### status
| Value    | Description                                        |
|----------|----------------------------------------------------|
| new      | Unassigned — New Messages tab mein dikhegi         |
| assigned | Assigned to agent — Assigned To Me tab mein dikhegi|
| resolved | Closed — Resolved Messages tab mein dikhegi        |

### priority
| Value  | First Response | Resolution | Color  |
|--------|----------------|------------|--------|
| urgent | 15 min         | 4 hours    | Red    |
| high   | 30 min         | 8 hours    | Orange |
| normal | 60 min         | 24 hours   | Blue   |
| low    | 240 min        | 48 hours   | Grey   |

### source_type
| Value          | Description                              |
|----------------|------------------------------------------|
| direct_message | Customer ne directly DM kiya             |
| comment_to_dm  | Post comment se Move to DM kiya gaya     |
| ad_click       | Click-to-message ad se aaya              |
| website_chat   | Website chat link se aaya                |

---

## Indexes

```sql
-- New Messages tab: tenant + status = 'new'
CREATE INDEX idx_conv_tenant_status ON conversations(tenant_id, status);

-- Assigned To Me tab: tenant + assigned_to + status
CREATE INDEX idx_conv_tenant_assigned_status ON conversations(tenant_id, assigned_to, status);

-- Customer history: tenant + customer
CREATE INDEX idx_conv_tenant_customer ON conversations(tenant_id, customer_id);

-- Page filter: tenant + page + status
CREATE INDEX idx_conv_tenant_page_status ON conversations(tenant_id, page_id, status);

-- SLA ordering (overdue pehle)
CREATE INDEX idx_conv_sla_due ON conversations(status, sla_resolution_due);

-- Ticket number search
CREATE UNIQUE INDEX idx_conv_ticket_number ON conversations(ticket_number);

-- List ordering by latest message
CREATE INDEX idx_conv_last_message ON conversations(tenant_id, last_message_at DESC);

-- External thread lookup (incoming webhook match)
CREATE INDEX idx_conv_external_thread ON conversations(external_thread_id, channel);
```

---

## Business Rules

- `status = 'new'` aur `assigned_to = NULL` → reply locked
- `status = 'assigned'` → reply unlocked, reason/label/about editable
- `status = 'resolved'` → reply locked, profile read-only
- Resolve karne se pehle `reason_id` mandatory hai
- Same `customer_id + channel + page_id` combination ka ek hi `status != 'resolved'` conversation ho sakta hai
- Reopen karne pe: assignee hai → `status = 'assigned'`, nahi hai → `status = 'new'`
- `csat_rating` 1-5 ke beech hona chahiye (CHECK constraint)
- `auto_resolved = TRUE` tab hoga jab customer ne CSAT request ka koi reply nahi diya aur timer expire ho gaya

---

## linked_order_data JSONB Structure

```json
{
  "order_id": "1078812",
  "platform": "Shopify",
  "order_date": "2026-08-24",
  "amount": 2426.00,
  "currency": "INR"
}
```

---

## Relationships

| Related Entity       | Type     | Via                            |
|----------------------|----------|--------------------------------|
| tenants              | Many-One | conversations.tenant_id        |
| customers            | Many-One | conversations.customer_id      |
| pages                | Many-One | conversations.page_id          |
| agents               | Many-One | conversations.assigned_to      |
| agents               | Many-One | conversations.resolved_by      |
| reasons              | Many-One | conversations.reason_id        |
| messages             | One-Many | messages.conversation_id       |
| assignment_history   | One-Many | assignment_history.conversation_id |
| conversation_labels  | One-Many | conversation_labels.conversation_id |
| shared_media         | One-Many | shared_media.conversation_id   |
