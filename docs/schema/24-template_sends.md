# Entity: template_sends

## Purpose
Har baar jab koi template send hota hai uska complete log store karta hai. Teen jagah se template send ho sakta hai — agent chat window se manually, journey step se automatically, ya client event/endpoint trigger se. Ye table batata hai kab, kise, kis template ne, kaun se variables ke saath, aur status kya tha (sent/delivered/read/failed). Reporting aur debugging ke liye use hota hai.

---

## Table Definition

```sql
CREATE TABLE template_sends (
    id                   UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id          UUID            NOT NULL REFERENCES templates(id),
    template_language_id UUID            NOT NULL REFERENCES template_languages(id),
    tenant_id            UUID            NOT NULL REFERENCES tenants(id),
    ticket_id            UUID            REFERENCES tickets(id),
    journey_id           UUID            REFERENCES journeys(id),
    journey_step_id      UUID            REFERENCES journey_steps(id),
    customer_id          UUID            NOT NULL REFERENCES customers(id),
    channel              VARCHAR(20)     NOT NULL CHECK (channel IN ('whatsapp', 'instagram', 'facebook')),
    sent_at              TIMESTAMP       NOT NULL DEFAULT NOW(),
    status               VARCHAR(20)     NOT NULL DEFAULT 'sent'
                                         CHECK (status IN ('sent', 'delivered', 'read', 'failed')),
    sent_by              VARCHAR(20)     NOT NULL CHECK (sent_by IN ('agent', 'journey', 'event')),
    event_ref            VARCHAR(255),
    variable_values      JSONB
);
```

---

## Columns

| Column               | Type         | Nullable | Default           | Description                                                                 |
|----------------------|--------------|----------|-------------------|-----------------------------------------------------------------------------|
| id                   | UUID         | NO       | gen_random_uuid() | Primary key                                                                 |
| template_id          | UUID         | NO       | —                 | FK → templates. Kaun sa template bheja gaya                                 |
| template_language_id | UUID         | NO       | —                 | FK → template_languages. Kaun si language mein bheja gaya                  |
| tenant_id            | UUID         | NO       | —                 | FK → tenants. Kis tenant ne bheja                                           |
| ticket_id            | UUID         | YES      | NULL              | FK → tickets. Agar agent ne chat window se bheja to ticket ka reference     |
| journey_id           | UUID         | YES      | NULL              | FK → journeys. Agar journey step se bheja to journey ka reference           |
| journey_step_id      | UUID         | YES      | NULL              | FK → journey_steps. Exact step ka reference jo fire hua                     |
| customer_id          | UUID         | NO       | —                 | FK → customers. Kise bheja gaya                                             |
| channel              | VARCHAR(20)  | NO       | —                 | ENUM: whatsapp / instagram / facebook                                       |
| sent_at              | TIMESTAMP    | NO       | NOW()             | Jab message send hua                                                        |
| status               | VARCHAR(20)  | NO       | sent              | ENUM: sent / delivered / read / failed. Meta webhook se update hota hai     |
| sent_by              | VARCHAR(20)  | NO       | —                 | ENUM: agent / journey / event. Kisne trigger kiya                           |
| event_ref            | VARCHAR(255) | YES      | NULL              | Client event jo trigger tha. e.g. "order_placed:1078812"                    |
| variable_values      | JSONB        | YES      | NULL              | Actual variable values used at send time. e.g. {"1":"Rahul","2":"1078812"} |

---

## sent_by Values

| Value   | Meaning                                              | ticket_id | journey_id | event_ref |
|---------|------------------------------------------------------|-----------|------------|-----------|
| agent   | Agent ne chat window se manually bheja               | filled    | null       | null      |
| journey | Journey step se automatically bheja                  | null      | filled     | null      |
| event   | Client API event/endpoint ne trigger kiya            | null      | null       | filled    |

---

## Status Flow (updated via Meta webhook)

```
sent
  ↓ Meta delivers to user's device
delivered
  ↓ User opens the message
read
  ↓ (terminal state — no further update)

OR

sent
  ↓ Meta could not deliver
failed
  (terminal state)
```

---

## variable_values — JSONB Example

```json
{
  "1": "Rahul",
  "2": "1078812",
  "3": "₹2,497",
  "4": "30 Aug 2026"
}

Key   = variable_index (as string)
Value = actual value used when message was sent

This is stored for:
→ Audit — what exactly was sent to this customer
→ Debugging — if wrong value was filled
→ Reporting — which offers were sent at what value
```

---

## Business Rules

- Every template send — manual or automated — must be logged here
- status starts as 'sent' and updates via Meta delivery webhook
- ticket_id, journey_id, journey_step_id are all nullable
  but at least one of (ticket_id / journey_id / event_ref) should be present
- variable_values stores snapshot of values at send time
  (not a reference — actual values frozen at send moment)
- This table is append-only — no updates except status field
- status update comes from Meta webhook (delivered / read / failed)

---

## Indexes

```sql
-- Tenant ke saare sends fetch karne ke liye (reporting)
CREATE INDEX idx_tpl_sends_tenant_id ON template_sends(tenant_id);

-- Template ke saare sends
CREATE INDEX idx_tpl_sends_template_id ON template_sends(template_id);

-- Customer ke saare sends
CREATE INDEX idx_tpl_sends_customer_id ON template_sends(customer_id);

-- Journey ke saare sends
CREATE INDEX idx_tpl_sends_journey_id ON template_sends(journey_id);

-- Ticket ke saare sends
CREATE INDEX idx_tpl_sends_ticket_id ON template_sends(ticket_id);

-- Time based reporting
CREATE INDEX idx_tpl_sends_sent_at ON template_sends(tenant_id, sent_at);
```

---

## Relationships

| Related Entity      | Type     | Via                              |
|---------------------|----------|----------------------------------|
| templates           | Many-One | template_sends.template_id       |
| template_languages  | Many-One | template_sends.template_language_id |
| tenants             | Many-One | template_sends.tenant_id         |
| tickets             | Many-One | template_sends.ticket_id         |
| journeys            | Many-One | template_sends.journey_id        |
| journey_steps       | Many-One | template_sends.journey_step_id   |
| customers           | Many-One | template_sends.customer_id       |
