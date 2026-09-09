# Entity: customers

## Purpose
End users jo tenant ke Meta pages pe message karte hain. Ek customer ek specific channel + platform user ID se identify hota hai. Same real person ke alag channels pe alag customer records honge (channel isolation rule).

---

## Table Definition

```sql
CREATE TABLE customers (
    id          UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID            NOT NULL REFERENCES tenants(id),
    external_id VARCHAR(255)    NOT NULL,
    channel     VARCHAR(50)     NOT NULL,
    username    VARCHAR(255),
    full_name   VARCHAR(255),
    avatar_url  TEXT,
    phone       VARCHAR(20),
    email       VARCHAR(255),
    profile_url TEXT,
    created_at  TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP
);
```

---

## Columns

| Column      | Type         | Nullable | Default           | Description                                        |
|-------------|--------------|----------|-------------------|----------------------------------------------------|
| id          | UUID         | NO       | gen_random_uuid() | Primary key                                        |
| tenant_id   | UUID         | NO       | —                 | FK → tenants.id                                    |
| external_id | VARCHAR(255) | NO       | —                 | Platform ka user ID (Instagram PSID, WA number etc)|
| channel     | VARCHAR(50)  | NO       | —                 | Konse platform se aaya                             |
| username    | VARCHAR(255) | YES      | NULL              | Platform handle (e.g. @football_hashir)            |
| full_name   | VARCHAR(255) | YES      | NULL              | Display name                                       |
| avatar_url  | TEXT         | YES      | NULL              | Profile picture                                    |
| phone       | VARCHAR(20)  | YES      | NULL              | Phone number (WhatsApp se available hota hai)      |
| email       | VARCHAR(255) | YES      | NULL              | Email (client ke CRM se link hone pe)              |
| profile_url | TEXT         | YES      | NULL              | IG/FB profile link (View Profile ke liye)          |
| created_at  | TIMESTAMP    | NO       | NOW()             | Record creation time                               |
| updated_at  | TIMESTAMP    | YES      | NULL              | Last profile update                                |

---

## Allowed Values

### channel
| Value     | Description   |
|-----------|---------------|
| instagram | Instagram DM  |
| facebook  | Facebook DM   |
| whatsapp  | WhatsApp      |

---

## Indexes

```sql
-- Tenant wise customer list
CREATE INDEX idx_customers_tenant_id ON customers(tenant_id);

-- Incoming message pe customer identify karne ke liye (most critical lookup)
CREATE UNIQUE INDEX idx_customers_external_channel_tenant 
    ON customers(external_id, channel, tenant_id);

-- Customer name search (Full Text Search)
CREATE INDEX idx_customers_full_name ON customers 
    USING GIN (to_tsvector('english', COALESCE(full_name, '') || ' ' || COALESCE(username, '')));
```

---

## Business Rules

- `external_id + channel + tenant_id` combination unique hoga
- Same real person WhatsApp se aaye → alag record, Instagram se aaye → alag record
- Phone/email client ke Core DB se link hone ke baad fill hoti hai
- Customer 360 data (orders, lifetime spend, tickets count) is table se nahi aata — client ke Core DB API se aata hai, yahan sirf identity store hoti hai

---

## Relationships

| Related Entity | Type     | Via                       |
|----------------|----------|---------------------------|
| tenants        | Many-One | customers.tenant_id       |
| conversations  | One-Many | conversations.customer_id |
