# Entity: messages

## Purpose
Ek conversation ke andar har ek message ka record. Customer ke messages, agent ke replies, internal notes, CSAT requests, aur commerce cards (product/payment/order tracking) sab yahan store hote hain.

---

## Table Definition

```sql
CREATE TABLE messages (
    id                      UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id         UUID            NOT NULL REFERENCES conversations(id),
    tenant_id               UUID            NOT NULL REFERENCES tenants(id),
    sender_type             VARCHAR(20)     NOT NULL,
    sender_id               UUID,
    message_type            VARCHAR(50)     NOT NULL DEFAULT 'text',
    content                 TEXT,
    payload                 JSONB,
    is_internal_note        BOOLEAN         NOT NULL DEFAULT FALSE,
    is_visible_to_customer  BOOLEAN         NOT NULL DEFAULT TRUE,
    quoted_message_id       UUID            REFERENCES messages(id),
    external_msg_id         VARCHAR(255),
    sent_at                 TIMESTAMP       NOT NULL DEFAULT NOW(),
    is_deleted              BOOLEAN         NOT NULL DEFAULT FALSE
);
```

---

## Columns

| Column                 | Type        | Nullable | Default           | Description                                      |
|------------------------|-------------|----------|-------------------|--------------------------------------------------|
| id                     | UUID        | NO       | gen_random_uuid() | Primary key                                      |
| conversation_id        | UUID        | NO       | —                 | FK → conversations.id                            |
| tenant_id              | UUID        | NO       | —                 | FK → tenants.id (quick filter)                   |
| sender_type            | VARCHAR(20) | NO       | —                 | Kisne bheja                                      |
| sender_id              | UUID        | YES      | NULL              | Agent ka id (agar agent ne bheja)                |
| message_type           | VARCHAR(50) | NO       | 'text'            | Message ka format/type                           |
| content                | TEXT        | YES      | NULL              | Plain text content                               |
| payload                | JSONB       | YES      | NULL              | Rich content (images, cards etc.)                |
| is_internal_note       | BOOLEAN     | NO       | FALSE             | TRUE = sirf agents ko dikhega                    |
| is_visible_to_customer | BOOLEAN     | NO       | TRUE              | FALSE = internal only                            |
| quoted_message_id      | UUID        | YES      | NULL              | FK → messages.id (quoted/reply-to message)       |
| external_msg_id        | VARCHAR(255)| YES      | NULL              | Platform ka message ID                           |
| sent_at                | TIMESTAMP   | NO       | NOW()             | Message send time                                |
| is_deleted             | BOOLEAN     | NO       | FALSE             | Soft delete                                      |

---

## Allowed Values

### sender_type
| Value    | Description                          |
|----------|--------------------------------------|
| customer | Customer ka message                  |
| agent    | Agent ka reply                       |
| system   | Auto-generated (CSAT request, etc.)  |
| bot      | AI/Chatbot ka message                |

### message_type
| Value                | Visible to Customer | Description                         |
|----------------------|---------------------|-------------------------------------|
| text                 | YES                 | Plain text message                  |
| image                | YES                 | Image attachment                    |
| video                | YES                 | Video attachment                    |
| pdf                  | YES                 | PDF document                        |
| voice_note           | YES                 | Audio message                       |
| order_tracking_card  | YES                 | Order status card                   |
| payment_link_card    | YES                 | Payment link card                   |
| product_card         | YES                 | Product details card                |
| csat_request         | YES                 | "Was this helpful?" message         |
| internal_note        | NO                  | Agent ka internal comment           |

---

## payload JSONB Structures

### image / video / pdf
```json
{
  "url": "https://cdn.example.com/file.jpg",
  "thumbnail_url": "https://cdn.example.com/thumb.jpg",
  "file_name": "receipt.pdf",
  "file_size": 204800,
  "mime_type": "image/jpeg"
}
```

### order_tracking_card
```json
{
  "order_id": "1078812",
  "courier": "Delhivery",
  "awb": "DL88372910145",
  "status": "Out for delivery",
  "tracking_url": "https://track.delhivery.com/..."
}
```

### payment_link_card
```json
{
  "amount": 1299.00,
  "currency": "INR",
  "reference": "PAY-2026-09-001",
  "expiry": "2026-09-10T14:00:00Z",
  "payment_url": "https://pay.example.com/..."
}
```

### product_card
```json
{
  "product_id": "PROD-001",
  "name": "Vitamin C 20% Face Serum",
  "sku": "VC-20-30ML",
  "price": 799.00,
  "currency": "INR",
  "image_url": "https://cdn.example.com/product.jpg",
  "buy_link": "https://store.example.com/product/..."
}
```

### csat_request
```json
{
  "question": "Was this helpful?",
  "options": ["helpful", "not_helpful"]
}
```

---

## Indexes

```sql
-- Conversation ki messages chronologically fetch karne ke liye
CREATE INDEX idx_messages_conv_sent ON messages(conversation_id, sent_at);

-- Tenant + conversation filter
CREATE INDEX idx_messages_tenant_conv ON messages(tenant_id, conversation_id);

-- Sender type filter (customer ke messages alag, agent ke alag)
CREATE INDEX idx_messages_sender_type ON messages(sender_type, conversation_id);

-- Internal notes alag filter karne ke liye
CREATE INDEX idx_messages_internal_note ON messages(is_internal_note, conversation_id);

-- Shared media fetch karne ke liye (images/videos wale messages)
CREATE INDEX idx_messages_type ON messages(message_type, conversation_id);
```

---

## Business Rules

- `is_internal_note = TRUE` hone pe `is_visible_to_customer = FALSE` automatically hoga
- Internal notes sirf `status = 'assigned'` conversations mein create ho sakte hain
- Customer reply tab possible hai jab `conversation.status = 'assigned'`
- `quoted_message_id` self-referential FK hai — ek message kisi doosre message ka quote ho sakta hai
- `is_deleted = TRUE` se message hide hoga lekin record preserve hoga
- CSAT request message `sender_type = 'system'` aur `message_type = 'csat_request'` hoga
- Ek conversation mein sirf ek active CSAT request hogi

---

## Relationships

| Related Entity | Type     | Via                          |
|----------------|----------|------------------------------|
| conversations  | Many-One | messages.conversation_id     |
| tenants        | Many-One | messages.tenant_id           |
| agents         | Many-One | messages.sender_id           |
| messages       | Many-One | messages.quoted_message_id   |
| shared_media   | One-Many | shared_media.message_id      |
