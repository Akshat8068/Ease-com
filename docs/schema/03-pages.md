# Entity: pages

## Purpose
Tenant ke connected Meta pages (Instagram, Facebook, WhatsApp). Ek tenant ke multiple pages ho sakte hain. Chat Inbox mein page dropdown se filter hota hai — page ke hisaab se conversations, posts sab alag alag dikhte hain.

---

## Table Definition

```sql
CREATE TABLE pages (
    id               UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id        UUID            NOT NULL REFERENCES tenants(id),
    external_page_id VARCHAR(255)    NOT NULL,
    name             VARCHAR(255)    NOT NULL,
    channel          VARCHAR(50)     NOT NULL,
    avatar_url       TEXT,
    is_active        BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at       TIMESTAMP       NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column           | Type         | Nullable | Default           | Description                                  |
|------------------|--------------|----------|-------------------|----------------------------------------------|
| id               | UUID         | NO       | gen_random_uuid() | Primary key                                  |
| tenant_id        | UUID         | NO       | —                 | FK → tenants.id                              |
| external_page_id | VARCHAR(255) | NO       | —                 | Meta platform ka page ID                     |
| name             | VARCHAR(255) | NO       | —                 | Page display name (e.g. "The Himalayan Organics") |
| channel          | VARCHAR(50)  | NO       | —                 | Platform channel                             |
| avatar_url       | TEXT         | YES      | NULL              | Page profile picture                         |
| is_active        | BOOLEAN      | NO       | TRUE              | Inactive pages hide ho jaayengi dropdown se  |
| created_at       | TIMESTAMP    | NO       | NOW()             | Record creation time                         |

---

## Allowed Values

### channel
| Value     | Description              |
|-----------|--------------------------|
| instagram | Instagram business page  |
| facebook  | Facebook page            |
| whatsapp  | WhatsApp Business number |

---

## Indexes

```sql
-- Tenant ki saari pages fetch karne ke liye
CREATE INDEX idx_pages_tenant_id ON pages(tenant_id);

-- Integration service se incoming webhook match karne ke liye
CREATE UNIQUE INDEX idx_pages_external_channel ON pages(external_page_id, channel);
```

---

## Business Rules

- Ek `external_page_id + channel` combination globally unique hoga
- Pages integration service se sync hote hain — manually create nahi hote
- `is_active = FALSE` se page dropdown mein nahi dikhega
- Conversations aur posts dono page se linked hote hain

---

## Relationships

| Related Entity | Type     | Via                    |
|----------------|----------|------------------------|
| tenants        | Many-One | pages.tenant_id        |
| conversations  | One-Many | conversations.page_id  |
| posts          | One-Many | posts.page_id          |
