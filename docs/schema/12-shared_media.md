# Entity: shared_media

## Purpose
Conversation mein share ki gayi saari media files ka index. Customer ya agent jo bhi image, video, PDF bheje — sab yahan record hota hai. Profile panel ke "Shared Media" section mein yahi data dikhta hai.

---

## Table Definition

```sql
CREATE TABLE shared_media (
    id              UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID            NOT NULL REFERENCES conversations(id),
    message_id      UUID            NOT NULL REFERENCES messages(id),
    tenant_id       UUID            NOT NULL REFERENCES tenants(id),
    media_type      VARCHAR(50)     NOT NULL,
    url             TEXT            NOT NULL,
    thumbnail_url   TEXT,
    file_name       VARCHAR(255),
    file_size       INTEGER,
    mime_type       VARCHAR(100),
    created_at      TIMESTAMP       NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column          | Type         | Nullable | Default           | Description                               |
|-----------------|--------------|----------|-------------------|-------------------------------------------|
| id              | UUID         | NO       | gen_random_uuid() | Primary key                               |
| conversation_id | UUID         | NO       | —                 | FK → conversations.id                     |
| message_id      | UUID         | NO       | —                 | FK → messages.id (source message)         |
| tenant_id       | UUID         | NO       | —                 | FK → tenants.id                           |
| media_type      | VARCHAR(50)  | NO       | —                 | File category                             |
| url             | TEXT         | NO       | —                 | Full file URL (CDN/storage)               |
| thumbnail_url   | TEXT         | YES      | NULL              | Preview thumbnail URL                     |
| file_name       | VARCHAR(255) | YES      | NULL              | Original file name                        |
| file_size       | INTEGER      | YES      | NULL              | File size in bytes                        |
| mime_type       | VARCHAR(100) | YES      | NULL              | MIME type (e.g. image/jpeg, video/mp4)    |
| created_at      | TIMESTAMP    | NO       | NOW()             | When media was shared                     |

---

## Allowed Values

### media_type
| Value | Description       | Examples                          |
|-------|-------------------|-----------------------------------|
| image | Image files       | jpg, png, gif, webp               |
| video | Video files       | mp4, mov                          |
| pdf   | PDF documents     | .pdf                              |
| audio | Audio/voice notes | mp3, ogg, m4a                     |

---

## Indexes

```sql
-- Profile panel ke "Shared Media" section ke liye
CREATE INDEX idx_shared_media_conv_id ON shared_media(conversation_id);

-- Tenant + conversation filter
CREATE INDEX idx_shared_media_tenant_conv ON shared_media(tenant_id, conversation_id);
```

---

## Business Rules

- Har media message ke saath ek row insert hogi (message save hone pe simultaneously)
- Internal note mein share ki gayi media bhi yahan store hogi (is_visible_to_customer = FALSE wali)
- `file_size` bytes mein hoga
- `thumbnail_url` images aur videos ke liye generate hogi
- PDFs ke liye `thumbnail_url` first page ka preview hoga (agar available)
- Media files CDN/object storage (S3 etc.) pe store hongi — yahan sirf URL reference hai

---

## Relationships

| Related Entity | Type     | Via                           |
|----------------|----------|-------------------------------|
| conversations  | Many-One | shared_media.conversation_id  |
| messages       | Many-One | shared_media.message_id       |
| tenants        | Many-One | shared_media.tenant_id        |
