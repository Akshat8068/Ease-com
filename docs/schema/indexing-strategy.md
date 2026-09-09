# Indexing Strategy

## Why Indexing Critical Hai

10 lakh+ conversations, multiple tenants, real-time queries — bina proper indexing ke:
- List load → 10+ seconds
- Search → timeout
- SLA ordering → full table scan

---

## Rule #1 — Multi-Tenant Filter (Sabse Zaroori)

**Har query mein `tenant_id` pehla filter hoga.**

```sql
-- WRONG (full table scan)
SELECT * FROM conversations WHERE status = 'new';

-- RIGHT (tenant-scoped)
SELECT * FROM conversations 
WHERE tenant_id = :tid AND status = 'new';
```

Isliye **har table mein `tenant_id` pe index** hai.

---

## Table-wise Index Details

### tenants
```sql
CREATE UNIQUE INDEX idx_tenants_slug ON tenants(slug);
```
| Index | Purpose |
|-------|---------|
| slug (UNIQUE) | Login/routing pe tenant lookup |

---

### agents
```sql
CREATE INDEX idx_agents_tenant_id ON agents(tenant_id);
CREATE UNIQUE INDEX idx_agents_email_tenant ON agents(email, tenant_id);
```
| Index | Purpose |
|-------|---------|
| tenant_id | Tenant ke saare agents fetch |
| email + tenant_id (UNIQUE) | Login authentication |

---

### pages
```sql
CREATE INDEX idx_pages_tenant_id ON pages(tenant_id);
CREATE UNIQUE INDEX idx_pages_external_channel ON pages(external_page_id, channel);
```
| Index | Purpose |
|-------|---------|
| tenant_id | Dropdown mein pages list |
| external_page_id + channel (UNIQUE) | Incoming webhook se page match |

---

### customers
```sql
CREATE INDEX idx_customers_tenant_id ON customers(tenant_id);
CREATE UNIQUE INDEX idx_customers_external_channel_tenant 
    ON customers(external_id, channel, tenant_id);
CREATE INDEX idx_customers_full_name 
    ON customers USING GIN (
        to_tsvector('english', 
            COALESCE(full_name, '') || ' ' || COALESCE(username, ''))
    );
```
| Index | Purpose |
|-------|---------|
| tenant_id | Tenant ke customers |
| external_id + channel + tenant_id (UNIQUE) | Incoming message pe customer identify |
| full_name GIN (Full Text) | Search by name |

---

### conversations ⭐ Most Critical Table
```sql
-- New Messages tab
CREATE INDEX idx_conv_tenant_status 
    ON conversations(tenant_id, status);

-- Assigned To Me tab
CREATE INDEX idx_conv_tenant_assigned_status 
    ON conversations(tenant_id, assigned_to, status);

-- Customer history
CREATE INDEX idx_conv_tenant_customer 
    ON conversations(tenant_id, customer_id);

-- Page filter (dropdown se page select hone pe)
CREATE INDEX idx_conv_tenant_page_status 
    ON conversations(tenant_id, page_id, status);

-- SLA ordering — overdue pehle
CREATE INDEX idx_conv_sla_due 
    ON conversations(status, sla_resolution_due ASC);

-- Ticket number search/lookup
CREATE UNIQUE INDEX idx_conv_ticket_number 
    ON conversations(ticket_number);

-- List ordering by latest message
CREATE INDEX idx_conv_last_message 
    ON conversations(tenant_id, last_message_at DESC);

-- Incoming webhook se open conversation find karna
CREATE INDEX idx_conv_external_thread 
    ON conversations(external_thread_id, channel);
```

| Index | Used By |
|-------|---------|
| tenant_id + status | New Messages list load |
| tenant_id + assigned_to + status | Assigned To Me list |
| tenant_id + customer_id | Customer history, 360 view |
| tenant_id + page_id + status | Page filter |
| status + sla_resolution_due | SLA ordering (overdue first) |
| ticket_number (UNIQUE) | Ticket search, URL routing |
| tenant_id + last_message_at | Default list sort |
| external_thread_id + channel | Webhook → existing ticket match |

---

### messages ⭐ High Volume Table
```sql
-- Chat window load (most frequent query)
CREATE INDEX idx_messages_conv_sent 
    ON messages(conversation_id, sent_at ASC);

-- Tenant scoped fetch
CREATE INDEX idx_messages_tenant_conv 
    ON messages(tenant_id, conversation_id);

-- Customer vs Agent messages filter
CREATE INDEX idx_messages_sender_type 
    ON messages(sender_type, conversation_id);

-- Internal notes filter
CREATE INDEX idx_messages_internal_note 
    ON messages(is_internal_note, conversation_id);

-- Shared media ke liye (image/video type messages)
CREATE INDEX idx_messages_type 
    ON messages(message_type, conversation_id);
```

| Index | Used By |
|-------|---------|
| conversation_id + sent_at | Chat window message list |
| tenant_id + conversation_id | Tenant scoped queries |
| sender_type + conversation_id | Customer/agent message filter |
| is_internal_note + conversation_id | Internal notes toggle |
| message_type + conversation_id | Shared media fetch |

---

### assignment_history
```sql
CREATE INDEX idx_assign_history_conv 
    ON assignment_history(conversation_id, created_at ASC);
CREATE INDEX idx_assign_history_tenant_agent 
    ON assignment_history(tenant_id, assigned_to);
```
| Index | Purpose |
|-------|---------|
| conversation_id + created_at | History trail chronologically |
| tenant_id + assigned_to | Agent ki saari assignments |

---

### labels
```sql
CREATE INDEX idx_labels_tenant_id ON labels(tenant_id);
CREATE UNIQUE INDEX idx_labels_name_tenant ON labels(name, tenant_id);
```

---

### conversation_labels
```sql
CREATE INDEX idx_conv_labels_conv_id ON conversation_labels(conversation_id);
CREATE INDEX idx_conv_labels_label_id ON conversation_labels(label_id);
CREATE UNIQUE INDEX idx_conv_labels_unique ON conversation_labels(conversation_id, label_id);
```
| Index | Purpose |
|-------|---------|
| conversation_id | Ek conversation ke labels |
| label_id | Ek label ki saari conversations (filter) |
| conv + label (UNIQUE) | Duplicate prevention |

---

### reasons
```sql
CREATE INDEX idx_reasons_tenant_id ON reasons(tenant_id);
CREATE INDEX idx_reasons_tenant_dept ON reasons(tenant_id, department);
CREATE UNIQUE INDEX idx_reasons_name_tenant ON reasons(name, tenant_id);
```
| Index | Purpose |
|-------|---------|
| tenant_id + department | Department-grouped dropdown |

---

### saved_replies
```sql
CREATE INDEX idx_saved_replies_tenant_id ON saved_replies(tenant_id);
CREATE INDEX idx_saved_replies_name_tenant ON saved_replies(name, tenant_id);
```

---

### shared_media
```sql
CREATE INDEX idx_shared_media_conv_id ON shared_media(conversation_id);
CREATE INDEX idx_shared_media_tenant_conv ON shared_media(tenant_id, conversation_id);
```

---

### posts
```sql
CREATE INDEX idx_posts_tenant_page ON posts(tenant_id, page_id);
CREATE UNIQUE INDEX idx_posts_external_id ON posts(external_post_id);
CREATE INDEX idx_posts_tenant_posted_at ON posts(tenant_id, posted_at DESC);
```

---

### post_comments
```sql
CREATE INDEX idx_post_comments_post_id ON post_comments(post_id);
CREATE INDEX idx_post_comments_parent ON post_comments(parent_comment_id);
CREATE INDEX idx_post_comments_tenant_post ON post_comments(tenant_id, post_id);
CREATE UNIQUE INDEX idx_post_comments_external_id ON post_comments(external_comment_id);
```

---

## Indexing Don'ts

```
❌ SELECT * FROM conversations WHERE status = 'new'
   (No tenant_id filter — full table scan)

❌ SELECT * FROM messages WHERE content LIKE '%query%'
   (LIKE with leading wildcard — no index use)
   → Full text search use karo

❌ Unnecessary indexes mat banao
   (Write operations slow ho jaati hain)
```

---

## Future Considerations (Scale ke saath)

| Scenario | Solution |
|----------|----------|
| 10M+ messages | Table partitioning by tenant_id + month |
| Global search | Elasticsearch / PostgreSQL FTS |
| Real-time updates | Redis pub/sub + WebSocket |
| SLA cron jobs | Redis sorted sets for timers |
| Frequently read data | Redis cache (conversation list, agent list) |
