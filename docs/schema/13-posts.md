# Entity: posts

## Purpose
Tenant ke Meta pages (Instagram/Facebook) ki social media posts. Integration service se sync hoti hain. Post's Comment tab mein left panel mein ye posts list hoti hain — agent inke comments manage karta hai.

---

## Table Definition

```sql
CREATE TABLE posts (
    id               UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id        UUID            NOT NULL REFERENCES tenants(id),
    page_id          UUID            NOT NULL REFERENCES pages(id),
    external_post_id VARCHAR(255)    NOT NULL UNIQUE,
    channel          VARCHAR(50)     NOT NULL,
    caption          TEXT,
    media_type       VARCHAR(50),
    media_url        TEXT,
    post_url         TEXT,
    like_count       INTEGER         NOT NULL DEFAULT 0,
    comment_count    INTEGER         NOT NULL DEFAULT 0,
    posted_at        TIMESTAMP,
    fetched_at       TIMESTAMP       NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column           | Type         | Nullable | Default           | Description                                       |
|------------------|--------------|----------|-------------------|---------------------------------------------------|
| id               | UUID         | NO       | gen_random_uuid() | Primary key                                       |
| tenant_id        | UUID         | NO       | —                 | FK → tenants.id                                   |
| page_id          | UUID         | NO       | —                 | FK → pages.id (konse page ki post hai)            |
| external_post_id | VARCHAR(255) | NO       | —                 | Meta platform ka post ID (globally unique)        |
| channel          | VARCHAR(50)  | NO       | —                 | 'instagram' ya 'facebook'                         |
| caption          | TEXT         | YES      | NULL              | Post ka text/caption                              |
| media_type       | VARCHAR(50)  | YES      | NULL              | Post ka media format                              |
| media_url        | TEXT         | YES      | NULL              | Post image/video URL                              |
| post_url         | TEXT         | YES      | NULL              | Original post ka platform URL (View Post ke liye) |
| like_count       | INTEGER      | NO       | 0                 | Current like count (periodically synced)          |
| comment_count    | INTEGER      | NO       | 0                 | Current comment count (shown in post list)        |
| posted_at        | TIMESTAMP    | YES      | NULL              | Original post publish time                        |
| fetched_at       | TIMESTAMP    | NO       | NOW()             | Last sync time from integration service           |

---

## Allowed Values

### channel
| Value     | Description      |
|-----------|------------------|
| instagram | Instagram post   |
| facebook  | Facebook post    |

### media_type
| Value    | Description          |
|----------|----------------------|
| image    | Single image post    |
| video    | Video post           |
| carousel | Multiple images/reel |
| text     | Text only post       |

---

## Indexes

```sql
-- Tenant + page ke posts list ke liye
CREATE INDEX idx_posts_tenant_page ON posts(tenant_id, page_id);

-- External post ID se quick lookup (webhook match)
CREATE UNIQUE INDEX idx_posts_external_id ON posts(external_post_id);

-- Chronological ordering (latest posts pehle)
CREATE INDEX idx_posts_tenant_posted_at ON posts(tenant_id, posted_at DESC);
```

---

## Business Rules

- Posts manually create nahi hoti — integration service se sync hoti hain
- `external_post_id` globally unique hoga
- `comment_count` aur `like_count` periodically update hoti hain (background sync)
- Post's Comment tab mein page dropdown filter se page select hone ke baad us page ki saari posts list mein dikhti hain
- `post_url` se "View Post" button kaam karta hai
- Posts delete nahi hoti — agar platform se delete ho to `comment_count = 0` ho sakta hai

---

## Relationships

| Related Entity | Type     | Via                    |
|----------------|----------|------------------------|
| tenants        | Many-One | posts.tenant_id        |
| pages          | Many-One | posts.page_id          |
| post_comments  | One-Many | post_comments.post_id  |
