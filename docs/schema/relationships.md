# Entity Relationships

## Overview

Poore schema ka relationship map. Ye document backend developer ko joins aur foreign keys samajhne mein help karega.

---

## Relationship Diagram (Text)

```
tenants
  │
  ├──< agents
  │      └── (created_by) saved_replies
  │      └── (assigned_to / assigned_from / assigned_by) assignment_history
  │      └── (assigned_to / resolved_by) conversations
  │
  ├──< pages
  │      └──< posts
  │             └──< post_comments ──> post_comments (parent-child, self-ref)
  │                     └──> conversations (dm_conversation_id)
  │
  ├──< customers
  │      └──< conversations
  │
  ├──< conversations ──> pages
  │      │         ──> agents (assigned_to)
  │      │         ──> agents (resolved_by)
  │      │         ──> reasons
  │      │
  │      ├──< messages ──> messages (quoted_message_id, self-ref)
  │      │      └──< shared_media
  │      │
  │      ├──< assignment_history
  │      │
  │      └──< conversation_labels
  │                  └──> labels
  │
  ├──< labels
  │      └──< conversation_labels
  │
  ├──< reasons
  │
  └──< saved_replies
```

---

## All Foreign Keys

| Table               | Column               | References              | On Delete   |
|---------------------|----------------------|-------------------------|-------------|
| agents              | tenant_id            | tenants.id              | RESTRICT    |
| pages               | tenant_id            | tenants.id              | RESTRICT    |
| customers           | tenant_id            | tenants.id              | RESTRICT    |
| conversations       | tenant_id            | tenants.id              | RESTRICT    |
| conversations       | customer_id          | customers.id            | RESTRICT    |
| conversations       | page_id              | pages.id                | RESTRICT    |
| conversations       | assigned_to          | agents.id               | SET NULL    |
| conversations       | resolved_by          | agents.id               | SET NULL    |
| conversations       | reason_id            | reasons.id              | SET NULL    |
| messages            | conversation_id      | conversations.id        | CASCADE     |
| messages            | tenant_id            | tenants.id              | RESTRICT    |
| messages            | sender_id            | agents.id               | SET NULL    |
| messages            | quoted_message_id    | messages.id             | SET NULL    |
| assignment_history  | conversation_id      | conversations.id        | CASCADE     |
| assignment_history  | tenant_id            | tenants.id              | RESTRICT    |
| assignment_history  | assigned_from        | agents.id               | SET NULL    |
| assignment_history  | assigned_to          | agents.id               | SET NULL    |
| assignment_history  | assigned_by          | agents.id               | SET NULL    |
| labels              | tenant_id            | tenants.id              | RESTRICT    |
| conversation_labels | conversation_id      | conversations.id        | CASCADE     |
| conversation_labels | label_id             | labels.id               | CASCADE     |
| conversation_labels | tenant_id            | tenants.id              | RESTRICT    |
| reasons             | tenant_id            | tenants.id              | RESTRICT    |
| saved_replies       | tenant_id            | tenants.id              | RESTRICT    |
| saved_replies       | created_by           | agents.id               | SET NULL    |
| shared_media        | conversation_id      | conversations.id        | CASCADE     |
| shared_media        | message_id           | messages.id             | CASCADE     |
| shared_media        | tenant_id            | tenants.id              | RESTRICT    |
| posts               | tenant_id            | tenants.id              | RESTRICT    |
| posts               | page_id              | pages.id                | RESTRICT    |
| post_comments       | post_id              | posts.id                | CASCADE     |
| post_comments       | tenant_id            | tenants.id              | RESTRICT    |
| post_comments       | parent_comment_id    | post_comments.id        | SET NULL    |
| post_comments       | dm_conversation_id   | conversations.id        | SET NULL    |

---

## Cardinality Summary

| Relationship                               | Type         |
|--------------------------------------------|--------------|
| tenant → agents                            | 1 : Many     |
| tenant → pages                             | 1 : Many     |
| tenant → customers                         | 1 : Many     |
| tenant → conversations                     | 1 : Many     |
| tenant → labels                            | 1 : Many     |
| tenant → reasons                           | 1 : Many     |
| tenant → saved_replies                     | 1 : Many     |
| customer → conversations                   | 1 : Many     |
| page → conversations                       | 1 : Many     |
| agent → conversations (assigned_to)        | 1 : Many     |
| agent → assignment_history                 | 1 : Many     |
| conversation → messages                    | 1 : Many     |
| conversation → assignment_history          | 1 : Many     |
| conversation → conversation_labels         | 1 : Many     |
| conversation → shared_media               | 1 : Many     |
| conversations ↔ labels (via conv_labels)   | Many : Many  |
| message → message (quoted_message_id)      | Self-ref 1:1 |
| post_comment → post_comment (parent-child) | Self-ref 1:M |
| page → posts                               | 1 : Many     |
| post → post_comments                       | 1 : Many     |
| post_comment → conversation (Move to DM)   | 1 : 1        |

---

## Key Queries — Common Joins

### New Messages Tab (unassigned conversations)
```sql
SELECT c.*, cu.username, cu.avatar_url, cu.channel,
       m.content AS last_message, m.sent_at
FROM conversations c
JOIN customers cu ON cu.id = c.customer_id
LEFT JOIN messages m ON m.id = (
    SELECT id FROM messages 
    WHERE conversation_id = c.id 
    ORDER BY sent_at DESC LIMIT 1
)
WHERE c.tenant_id = :tenant_id
  AND c.page_id = :page_id
  AND c.status = 'new'
ORDER BY c.sla_resolution_due ASC, c.last_message_at DESC;
```

### Conversation History (History Icon)
```sql
SELECT c.ticket_number, c.created_at, c.resolved_at,
       a.name AS resolved_by_agent,
       r.name AS reason,
       c.linked_order_id,
       ARRAY_AGG(l.name) AS labels
FROM conversations c
LEFT JOIN agents a ON a.id = c.resolved_by
LEFT JOIN reasons r ON r.id = c.reason_id
LEFT JOIN conversation_labels cl ON cl.conversation_id = c.id
LEFT JOIN labels l ON l.id = cl.label_id
WHERE c.customer_id = :customer_id
  AND c.tenant_id = :tenant_id
  AND c.status = 'resolved'
GROUP BY c.id, a.name, r.name
ORDER BY c.created_at DESC;
```

### Assigned To Me Tab
```sql
SELECT c.*, cu.username, cu.avatar_url
FROM conversations c
JOIN customers cu ON cu.id = c.customer_id
WHERE c.tenant_id = :tenant_id
  AND c.assigned_to = :agent_id
  AND c.status = 'assigned'
ORDER BY c.sla_resolution_due ASC;
```
