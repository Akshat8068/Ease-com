# Entity: templates

## Purpose
WhatsApp (Meta) message templates jo agent create karta hai. Har template Meta se approve hona zaroori hai — approve hone ke baad hi kisi journey step mein, keyword rule mein, ya agent chat window se bheja ja sakta hai. Template ek tenant ke andar unique hota hai aur ek agent create karta hai lekin koi bhi agent use kar sakta hai.

---

## Table Definition

```sql
CREATE TABLE templates (
    id          UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID            NOT NULL REFERENCES tenants(id),
    name        VARCHAR(255)    NOT NULL,
    channel     VARCHAR(20)     NOT NULL CHECK (channel IN ('whatsapp', 'instagram', 'facebook')),
    category    VARCHAR(20)     NOT NULL CHECK (category IN ('utility', 'marketing', 'authentication')),
    is_active   BOOLEAN         NOT NULL DEFAULT TRUE,
    created_by  UUID            NOT NULL REFERENCES agents(id),
    created_at  TIMESTAMP       NOT NULL DEFAULT NOW(),
    updated_by  UUID            REFERENCES agents(id),
    updated_at  TIMESTAMP       NOT NULL DEFAULT NOW()
);
```

---

## Columns

| Column     | Type         | Nullable | Default           | Description                                                                 |
|------------|--------------|----------|-------------------|-----------------------------------------------------------------------------|
| id         | UUID         | NO       | gen_random_uuid() | Primary key                                                                 |
| tenant_id  | UUID         | NO       | —                 | FK → tenants. Kis tenant ka template hai                                    |
| name       | VARCHAR(255) | NO       | —                 | Template name — unique per tenant. e.g. "cart_reminder", "order_confirmation" |
| channel    | VARCHAR(20)  | NO       | —                 | ENUM: whatsapp / instagram / facebook. Currently only whatsapp for templates |
| category   | VARCHAR(20)  | NO       | —                 | ENUM: utility / marketing / authentication. Meta category — affects price + opt-in requirement |
| is_active  | BOOLEAN      | NO       | TRUE              | Soft delete — inactive templates dropdowns mein nahi dikhenge               |
| created_by | UUID         | NO       | —                 | FK → agents. Agent jisne template create kiya                               |
| created_at | TIMESTAMP    | NO       | NOW()             | Creation time                                                               |
| updated_by | UUID         | YES      | NULL              | FK → agents. Last agent jisne edit kiya                                     |
| updated_at | TIMESTAMP    | NO       | NOW()             | Last edit time                                                              |

---

## Category Types

| Category       | Meaning                                                                 |
|----------------|-------------------------------------------------------------------------|
| utility        | Transactional — order confirmation, shipping update. Cheaper, no opt-in needed |
| marketing      | Promotional — cart recovery, offers, new arrivals. Costlier, needs opt-in |
| authentication | OTP / verification messages only                                        |

---

## Business Rules

- Template name ek tenant ke andar UNIQUE hoga
- Template sirf `channel = whatsapp` ke liye abhi use hota hai (Meta templates)
- Template create karne ke baad Meta ko submit karna hoga review ke liye
- Sirf `approved` status wale templates journey steps mein use ho sakte hain
- Ek template ke multiple language variants ho sakte hain (template_languages table)
- Har language Meta se alag review hoti hai
- Template create karta hai ek agent, use kar sakta hai koi bhi agent
- `is_active = false` se template soft delete hoga — existing journeys unaffected

---

## Three Ways a Template is Sent

```
WAY 1 — Agent manually from chat window
→ Agent opens assigned ticket
→ Clicks send template
→ Picks from approved templates list
→ Variables auto-filled from order/customer data
→ Sends via Meta outbound API

WAY 2 — Journey step (automated)
→ Journey enrollment reaches a step
→ Step has template_id linked
→ System checks status = approved
→ Variables filled from client API event payload
→ Sends via Meta outbound API

WAY 3 — Client event/endpoint trigger
→ Client fires event to our system (e.g. order_shipped)
→ Our system matches to configured template
→ Variables filled from event payload
→ Sends via Meta outbound API
```

---

## Dependencies

| Depends on   | Why                                          |
|--------------|----------------------------------------------|
| tenants      | Every template belongs to one tenant         |
| agents       | created_by and updated_by tracking           |
| Meta API     | External — template must be submitted and approved by Meta before use |

---

## Child Tables

| Table                 | Purpose                                              |
|-----------------------|------------------------------------------------------|
| template_languages    | Per language variant — header, body, footer, status  |
| template_variables    | N sample values + data_key mapping per language      |
| template_buttons      | N buttons per language variant                       |
| template_sends        | Log of every send — who, when, from where            |

---

## Indexes

```sql
-- Tenant ke saare templates fetch karne ke liye
CREATE INDEX idx_templates_tenant_id ON templates(tenant_id);

-- Name uniqueness per tenant
CREATE UNIQUE INDEX idx_templates_name_tenant ON templates(name, tenant_id);
```

---

## Relationships

| Related Entity     | Type     | Via                       |
|--------------------|----------|---------------------------|
| tenants            | Many-One | templates.tenant_id       |
| agents             | Many-One | templates.created_by      |
| agents             | Many-One | templates.updated_by      |
| template_languages | One-Many | template_languages.template_id |
| journey_steps      | One-Many | journey_steps.template_id |
| template_sends     | One-Many | template_sends.template_id |
