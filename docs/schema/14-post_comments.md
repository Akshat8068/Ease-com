# Entity: post_comments

## Purpose
Social media posts ke comments aur unke nested replies. Post's Comment tab mein right pane mein ye dikhte hain. Agent teen actions le sakta hai: Reply (nested), Move to DM (New Messages mein ticket banta hai), Hide (platform pe hide).

---

## Table Definition

```sql
CREATE TABLE post_comments (
    id                    UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id               UUID            NOT NULL REFERENCES posts(id),
    tenant_id             UUID            NOT NULL REFERENCES tenants(id),
    external_comment_id   VARCHAR(255)    NOT NULL UNIQUE,
    parent_comment_id     UUID            REFERENCES post_comments(id),
    commenter_external_id VARCHAR(255)    NOT NULL,
    commenter_name        VARCHAR(255),
    commenter_avatar      TEXT,
    channel               VARCHAR(50)     NOT NULL,
    content               TEXT,
    media_url             TEXT,
    is_hidden             BOOLEAN         NOT NULL DEFAULT FALSE,
    moved_to_dm           BOOLEAN         NOT NULL DEFAULT FALSE,
    dm_conversation_id    UUID            REFERENCES conversations(id),
    commented_at          TIMESTAMP,
    fetched_at            TIMESTAMP       NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column                 | Type         | Nullable | Default           | Description                                         |
|------------------------|--------------|----------|-------------------|-----------------------------------------------------|
| id                     | UUID         | NO       | gen_random_uuid() | Primary key                                         |
| post_id                | UUID         | NO       | —                 | FK → posts.id                                       |
| tenant_id              | UUID         | NO       | —                 | FK → tenants.id                                     |
| external_comment_id    | VARCHAR(255) | NO       | —                 | Meta platform ka comment ID (globally unique)       |
| parent_comment_id      | UUID         | YES      | NULL              | FK → post_comments.id (NULL = parent comment)       |
| commenter_external_id  | VARCHAR(255) | NO       | —                 | Commenter ka platform user ID                       |
| commenter_name         | VARCHAR(255) | YES      | NULL              | Commenter ka display name                           |
| commenter_avatar       | TEXT         | YES      | NULL              | Commenter ka avatar URL                             |
| channel                | VARCHAR(50)  | NO       | —                 | 'instagram' ya 'facebook'                           |
| content                | TEXT         | YES      | NULL              | Comment text                                        |
| media_url              | TEXT         | YES      | NULL              | Attached media URL (agar koi image/video hai)       |
| is_hidden              | BOOLEAN      | NO       | FALSE             | TRUE = platform pe hidden hai                       |
| moved_to_dm            | BOOLEAN      | NO       | FALSE             | TRUE = DM ticket already bana hai                   |
| dm_conversation_id     | UUID         | YES      | NULL              | FK → conversations.id (Move to DM se bani ticket)   |
| commented_at           | TIMESTAMP    | YES      | NULL              | Original comment time                               |
| fetched_at             | TIMESTAMP    | NO       | NOW()             | Last sync time                                      |

---

## Allowed Values

### parent_comment_id
| Value | Meaning                                |
|-------|----------------------------------------|
| NULL  | Parent comment (top-level)             |
| UUID  | Nested reply (child of parent comment) |

---

## Indexes

```sql
-- Post ke saare comments fetch karne ke liye
CREATE INDEX idx_post_comments_post_id ON post_comments(post_id);

-- Nested replies fetch karne ke liye
CREATE INDEX idx_post_comments_parent ON post_comments(parent_comment_id);

-- Tenant + post filter
CREATE INDEX idx_post_comments_tenant_post ON post_comments(tenant_id, post_id);

-- External comment ID uniqueness
CREATE UNIQUE INDEX idx_post_comments_external_id ON post_comments(external_comment_id);
```

---

## Business Rules

- `parent_comment_id = NULL` → top-level parent comment
- `parent_comment_id = UUID` → nested reply
- **Move to DM sirf parent comments pe allowed hai** (nested replies pe nahi)
- Move to DM flow:
  1. `moved_to_dm = TRUE` set karo
  2. New conversation create karo (`conversations` table mein)
  3. `dm_conversation_id` = new conversation ka id
  4. `source_type = 'comment_to_dm'` conversation mein
  5. Commenter = customer, comment text = first message
- `is_hidden = TRUE` hone ke baad "Hide" button "Unhide" ban sakta hai (future feature)
- Ek parent comment ka sirf ek baar Move to DM ho sakta hai (`moved_to_dm = TRUE` check karo)
- Platform pe reply karne se ek nested reply row insert hogi (`parent_comment_id` set hoga)

---

## Move to DM — Flow Diagram

```
Parent Comment selected → "Move to DM" click
            ↓
post_comments.moved_to_dm = TRUE
post_comments.dm_conversation_id = new_conv_id
            ↓
conversations table mein insert:
  customer_id  = commenter ka customer record
  source_type  = 'comment_to_dm'
  status       = 'new'
  first message = comment text
            ↓
New Messages tab mein unassigned ticket dikhegi
```

---

## Relationships

| Related Entity | Type     | Via                                 |
|----------------|----------|-------------------------------------|
| posts          | Many-One | post_comments.post_id               |
| tenants        | Many-One | post_comments.tenant_id             |
| post_comments  | Many-One | post_comments.parent_comment_id     |
| conversations  | Many-One | post_comments.dm_conversation_id    |
